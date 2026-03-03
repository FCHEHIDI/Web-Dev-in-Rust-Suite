````markdown
# Projet 07 — gRPC avec Tonic + server streaming

## Contexte

Tu implémentes une API gRPC en Rust avec **Tonic** pour la communication service-à-service.  
gRPC est le protocole standard inter-services dans les architectures microservices : typé, binaire (protobuf), bidirectionnel, et HTTP/2.  
Là où REST/JSON est adapté pour les clients publics, gRPC est adapté pour les appels internes : 5–10x plus rapide, contrat fort entre services, streaming natif.

Ce projet couvre les 4 types d'appels gRPC : unaire, server streaming, client streaming, et bidirectionnel.

**Environnement** : Rust, `tonic` 0.11+, `prost` (protobuf), `tonic-build` (codegen), Tokio  
**Cas d'usage** : service de métriques temps réel streamé vers un client dashboard

---

## Objectifs du projet

1. Définir un **fichier `.proto`** complet avec messages, service, et streaming
2. Générer le **code Rust depuis le proto** via `tonic-build` dans `build.rs`
3. Implémenter le **serveur gRPC** avec les 4 types de RPC
4. Implémenter le **client gRPC** typé avec retry et timeout configurable
5. Implémenter l'**intercepteur** serveur : authentification par metadata (bearer token)
6. Implémenter la **réflexion gRPC** (permet d'utiliser `grpcurl` / Postman avec le service)

---

## Spécifications techniques

### Fichier proto

```protobuf
// proto/metrics.proto
syntax = "proto3";
package metrics.v1;

import "google/protobuf/timestamp.proto";

// --- Messages ---

message MetricPoint {
    string name        = 1;   // "cpu_usage", "memory_bytes", ...
    double value       = 2;
    map<string, string> labels = 3;
    google.protobuf.Timestamp timestamp = 4;
}

message QueryRequest {
    string metric_name = 1;
    int64 from_unix    = 2;
    int64 to_unix      = 3;
    int32 limit        = 4;
}

message RecordRequest {
    repeated MetricPoint points = 1;
}

message RecordResponse {
    int32 points_accepted = 1;
}

message WatchRequest {
    repeated string metric_names = 1;
    int32 interval_ms            = 2;   // fréquence de push
}

message StreamRequest {
    // client streaming — envoi batch de points
    MetricPoint point = 1;
}

message StreamSummary {
    int32 total_received = 1;
    int32 total_accepted = 2;
    repeated string errors = 3;
}

// --- Service ---

service MetricsService {
    // 1. Unaire : récupérer des points historiques
    rpc Query(QueryRequest) returns (stream MetricPoint);

    // 2. Server streaming : recevoir les métriques live
    rpc Watch(WatchRequest) returns (stream MetricPoint);

    // 3. Client streaming : envoyer un batch de points
    rpc RecordBatch(stream StreamRequest) returns (StreamSummary);

    // 4. Bidirectionnel : flush en temps réel avec ack
    rpc Sync(stream MetricPoint) returns (stream RecordResponse);
}
```

### build.rs

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .build_server(true)
        .build_client(true)
        .file_descriptor_set_path(
            std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap())
                .join("metrics_descriptor.bin")
        )
        .compile(&["proto/metrics.proto"], &["proto", "proto/google"])?;
    Ok(())
}
```

### Implémentation serveur

```rust
pub struct MetricsServiceImpl {
    store: Arc<MetricsStore>,
}

#[tonic::async_trait]
impl MetricsService for MetricsServiceImpl {
    // Server streaming : Watch
    type WatchStream = Pin<Box<dyn Stream<Item = Result<MetricPoint, Status>> + Send>>;

    async fn watch(
        &self,
        request: Request<WatchRequest>,
    ) -> Result<Response<Self::WatchStream>, Status> {
        let req = request.into_inner();
        let store = self.store.clone();

        let stream = async_stream::stream! {
            let mut interval = tokio::time::interval(
                Duration::from_millis(req.interval_ms as u64)
            );
            loop {
                interval.tick().await;
                for name in &req.metric_names {
                    match store.latest(name).await {
                        Ok(point) => yield Ok(point),
                        Err(e) => yield Err(Status::internal(e.to_string())),
                    }
                }
            }
        };

        Ok(Response::new(Box::pin(stream)))
    }

    // Client streaming : RecordBatch
    async fn record_batch(
        &self,
        request: Request<Streaming<StreamRequest>>,
    ) -> Result<Response<StreamSummary>, Status> {
        let mut stream = request.into_inner();
        let mut received = 0;
        let mut accepted = 0;

        while let Some(req) = stream.message().await? {
            received += 1;
            if let Err(e) = self.store.insert(req.point.unwrap()).await {
                tracing::warn!(error = %e, "point rejected");
            } else {
                accepted += 1;
            }
        }

        Ok(Response::new(StreamSummary {
            total_received: received,
            total_accepted: accepted,
            errors: vec![],
        }))
    }
}
```

### Intercepteur d'authentification

```rust
/// Middleware Tower pour gRPC : vérifie le bearer token dans les metadata
pub fn auth_interceptor(req: Request<()>) -> Result<Request<()>, Status> {
    let token = req
        .metadata()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "))
        .ok_or_else(|| Status::unauthenticated("missing authorization header"))?;

    validate_service_token(token)
        .map_err(|_| Status::unauthenticated("invalid token"))?;

    Ok(req)
}

// Enregistrement sur le serveur
let service = MetricsServiceServer::with_interceptor(
    MetricsServiceImpl::new(store),
    auth_interceptor,
);
```

### Client gRPC avec retry

```rust
pub struct MetricsClient {
    inner: metrics_service_client::MetricsServiceClient<
        tonic::codegen::InterceptedService<Channel, AuthInterceptor>
    >,
    config: ClientConfig,
}

impl MetricsClient {
    pub async fn connect(config: ClientConfig) -> Result<Self, ClientError> {
        let channel = Channel::from_shared(config.endpoint.clone())?
            .timeout(Duration::from_secs(5))
            .connect_timeout(Duration::from_secs(10))
            .connect()
            .await?;

        let inner = metrics_service_client::MetricsServiceClient::with_interceptor(
            channel,
            AuthInterceptor { token: config.token.clone() },
        );

        Ok(Self { inner, config })
    }

    /// Watch avec reconnexion automatique sur erreur.
    pub async fn watch_with_reconnect(
        &mut self,
        req: WatchRequest,
        mut on_point: impl FnMut(MetricPoint) + Send,
    ) {
        loop {
            match self.inner.watch(req.clone()).await {
                Ok(response) => {
                    let mut stream = response.into_inner();
                    while let Ok(Some(point)) = stream.message().await {
                        on_point(point);
                    }
                    tracing::info!("stream ended, reconnecting in 2s");
                }
                Err(e) => tracing::error!(error = %e, "watch failed"),
            }
            tokio::time::sleep(Duration::from_secs(2)).await;
        }
    }
}
```

---

## Livrables attendus

- [ ] `proto/metrics.proto` : service complet avec les 4 types de RPC
- [ ] `build.rs` : codegen tonic-build avec file descriptor pour la réflexion
- [ ] Serveur : Query (unaire), Watch (server stream), RecordBatch (client stream), Sync (bidi)
- [ ] `auth_interceptor` : validation bearer token sur toutes les requêtes
- [ ] Réflexion gRPC (`tonic-reflection`) : service introspectable via `grpcurl`
- [ ] Client avec reconnexion automatique sur `watch_with_reconnect`
- [ ] Tests : unaire, stream 10 points, batch 100 points, erreur d'auth → Status::Unauthenticated
- [ ] `docker-compose.yml` : serveur gRPC + Prometheus metrics sur `/metrics`

---

## Critères de qualité

- Le proto est la source de vérité — jamais de types définis manuellement qui dupliquent le proto
- Les streamings gérèrent proprement la déconnexion du client (pas de goroutine leak équivalent)
- `cargo clippy -- -D warnings` : zéro warning
- `grpcurl -plaintext localhost:50051 list` doit retourner le service via réflexion
- Chaque RPC a son span tracing avec les métadonnées gRPC (method, status code)

---

## Ressources

- Tonic documentation : https://docs.rs/tonic
- Exemples Tonic officiels : https://github.com/hyperium/tonic/tree/master/examples
- `async-stream` pour les server streams : https://docs.rs/async-stream
- `tonic-reflection` : https://docs.rs/tonic-reflection
- gRPC status codes : https://grpc.github.io/grpc/core/md_doc_statuscodes.html
````
