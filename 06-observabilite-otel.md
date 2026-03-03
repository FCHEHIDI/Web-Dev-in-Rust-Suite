````markdown
# Projet 06 — Observabilité : tracing, métriques & logs (OpenTelemetry)

## Contexte

Tu instrumentes une API Axum avec la stack d'observabilité standard de l'industrie : **OpenTelemetry** pour les traces distribuées, **Prometheus** pour les métriques, et **tracing structuré** pour les logs.  
Sans ces outils, un bug en production est un puzzle aveugle. Avec eux, tu as le contexte complet : quelle requête, quel user, quelle durée, quel chemin de code, quelle erreur.

Ce projet te donne les patterns utilisés dans tous les backends SaaS sérieux, compatibles avec Grafana, Jaeger, Datadog, et tout exporteur OTEL.

**Environnement** : Rust, Axum, `tracing`, `tracing-opentelemetry`, `opentelemetry`, `prometheus`, `metrics`  
**Exporteurs** : OTLP (Jaeger/Tempo) pour les traces, `/metrics` Prometheus pour les métriques

---

## Objectifs du projet

1. Configurer **tracing structuré** avec `tracing` + `tracing-subscriber` (JSON en prod, pretty en dev)
2. Instrumenter chaque handler avec des **spans automatiques** et des **champs de contexte** (user_id, request_id)
3. Exporter les traces vers **Jaeger via OTLP** avec propagation du contexte distribué (W3C TraceContext)
4. Exposer des **métriques Prometheus** sur `/metrics` : requêtes/sec, latences (histogrammes), erreurs par code
5. Implémenter des **métriques métier** custom : users_registered_total, jobs_processed_total
6. Implémenter le **health check enrichi** : état de chaque dépendance (DB, Redis) avec latence

---

## Spécifications techniques

### Structure du projet

```
observabilite-otel/
├── Cargo.toml
├── docker-compose.yml          ← Jaeger + Prometheus + Grafana
├── grafana/
│   └── dashboards/api.json     ← dashboard importable
├── src/
│   ├── main.rs
│   ├── telemetry/
│   │   ├── mod.rs
│   │   ├── tracing.rs          ← init tracing + OTLP exporter
│   │   ├── metrics.rs          ← registre Prometheus + métriques globales
│   │   └── health.rs           ← health check multi-dépendances
│   ├── middleware/
│   │   ├── trace_layer.rs      ← span par requête + champs auto
│   │   └── metrics_layer.rs    ← incrément compteurs Tower
│   ├── app.rs
│   └── routes/
│       ├── health.rs           ← GET /health, GET /health/ready
│       └── metrics_endpoint.rs ← GET /metrics (Prometheus scrape)
└── tests/
    └── telemetry_tests.rs
```

### Initialisation tracing

```rust
pub fn init_tracing(config: &TelemetryConfig) -> OtelGuard {
    // Formattage : JSON en prod (parseable par Loki/Splunk), pretty en dev
    let fmt_layer = tracing_subscriber::fmt::layer()
        .with_target(true)
        .with_thread_ids(false);

    let fmt_layer = if config.json_logs {
        fmt_layer.json().boxed()
    } else {
        fmt_layer.pretty().boxed()
    };

    // Exporter OTLP vers Jaeger / Tempo
    let otlp_exporter = opentelemetry_otlp::new_exporter()
        .tonic()
        .with_endpoint(&config.otlp_endpoint);

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(otlp_exporter)
        .with_trace_config(
            opentelemetry_sdk::trace::Config::default()
                .with_resource(Resource::new(vec![
                    KeyValue::new("service.name", config.service_name.clone()),
                    KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
                ]))
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(fmt_layer)
        .with(otel_layer)
        .init();

    OtelGuard  // shutdown OTEL proprement au Drop
}
```

### Métriques Prometheus

```rust
// Métriques globales — utilisables depuis n'importe quel module
pub struct ApiMetrics {
    pub http_requests_total:    CounterVec,    // labels: method, path, status
    pub http_request_duration:  HistogramVec,  // labels: method, path
    pub http_requests_in_flight: GaugeVec,     // labels: method

    // Métriques métier
    pub users_registered_total:  Counter,
    pub jobs_enqueued_total:     CounterVec,   // labels: job_type
    pub jobs_processed_total:    CounterVec,   // labels: job_type, status
    pub db_query_duration:       HistogramVec, // labels: query_name
}

lazy_static::lazy_static! {
    pub static ref METRICS: ApiMetrics = ApiMetrics::register();
}

// Exemple d'usage dans un handler
pub async fn register_user(/* ... */) -> Result<impl IntoResponse, AppError> {
    let _timer = METRICS.http_request_duration
        .with_label_values(&["POST", "/auth/register"])
        .start_timer();  // auto-observe au Drop

    // ... logique ...

    METRICS.users_registered_total.inc();
    Ok((StatusCode::CREATED, Json(response)))
}
```

### Middleware Tower pour métriques HTTP automatiques

```rust
pub struct MetricsLayer;

impl<S> Layer<S> for MetricsLayer {
    type Service = MetricsMiddleware<S>;
    fn layer(&self, inner: S) -> Self::Service { MetricsMiddleware { inner } }
}

// Implémentation : intercepte chaque requête/réponse, incrémente les compteurs
// Sans modifier les handlers — c'est la force du pattern Tower
impl<S, ReqBody, ResBody> Service<Request<ReqBody>> for MetricsMiddleware<S>
where
    S: Service<Request<ReqBody>, Response = Response<ResBody>>,
{
    // start_timer → call inner → observe duration + increment counter
}
```

### Spans instrumentés dans les handlers

```rust
// Les spans sont automatiques via TraceLayer, mais on peut enrichir avec des champs custom
#[tracing::instrument(
    name = "get_product",
    skip(state),                    // ne pas logger l'état complet
    fields(product_id = %id, user_id = tracing::field::Empty)
)]
pub async fn get_product(
    State(state): State<AppState>,
    auth: AuthUser,
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse, AppError> {
    // Ajouter le user_id au span courant (disponible après l'authentification)
    tracing::Span::current().record("user_id", &auth.id.to_string());

    let product = state.product_service.get_by_id(id)
        .await
        .map_err(|e| {
            tracing::error!(error = %e, "product lookup failed");  // loggué avec trace_id
            e
        })?;

    Ok(Json(ProductResponse::from(product)))
}
```

### Health check enrichi

```rust
#[derive(Serialize)]
pub struct HealthResponse {
    pub status: HealthStatus,
    pub version: &'static str,
    pub uptime_secs: u64,
    pub checks: HashMap<String, DependencyHealth>,
}

#[derive(Serialize)]
pub struct DependencyHealth {
    pub status: HealthStatus,
    pub latency_ms: Option<f64>,
    pub error: Option<String>,
}

#[derive(Serialize)]
#[serde(rename_all = "lowercase")]
pub enum HealthStatus { Ok, Degraded, Down }

// GET /health       → rapide, toujours 200 (liveness)
// GET /health/ready → vérifie DB + Redis, 503 si dégradé (readiness)
```

---

## Livrables attendus

- [ ] `telemetry/tracing.rs` : init OTEL avec OTLP exporter + JSON/pretty logs
- [ ] `telemetry/metrics.rs` : `ApiMetrics` global avec compteurs + histogrammes
- [ ] `middleware/metrics_layer.rs` : Tower layer metrics HTTP automatiques
- [ ] `middleware/trace_layer.rs` : enrichissement spans (request_id, user_id)
- [ ] `routes/metrics_endpoint.rs` : `/metrics` scrappable par Prometheus
- [ ] `routes/health.rs` : `/health` (liveness) + `/health/ready` (readiness)
- [ ] `docker-compose.yml` : Jaeger + Prometheus + Grafana opérationnels
- [ ] Dashboard Grafana importable avec RPS, latences p50/p95/p99, error rate
- [ ] Tests : vérifier que les métriques s'incrémentent après les requêtes

---

## Critères de qualité

- Chaque log contient : `trace_id`, `span_id`, `request_id`, `user_id` si authentifié
- Les traces OTLP arrivent dans Jaeger visible sur `http://localhost:16686`
- Les histogrammes Prometheus ont des buckets appropriés : `[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]`
- `cargo clippy -- -D warnings` : zéro warning
- Les metrics ne bloquent jamais le handler (registre global non-bloquant)

---

## Ressources

- `tracing` ecosystem : https://docs.rs/tracing
- `opentelemetry-otlp` : https://docs.rs/opentelemetry-otlp
- `tracing-opentelemetry` : https://docs.rs/tracing-opentelemetry
- `prometheus` crate : https://docs.rs/prometheus
- Guide Axum observabilité : https://github.com/tokio-rs/axum/blob/main/examples/tracing-aka-logging/
````
