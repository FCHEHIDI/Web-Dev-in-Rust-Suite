````markdown
# Projet 08 — API Gateway / Reverse proxy service mesh

## Contexte

Tu implémentes un **API Gateway** en Rust avec Axum + Tower : point d'entrée unique qui route, transforme, et protège les requêtes vers plusieurs services backend.  
C'est l'architecture de production des plateformes SaaS avec plus de 2–3 services : Kong, Traefik, et AWS API Gateway font ce travail côté managed, mais comprendre comment les implémenter en Rust consolide tous les patterns Tower/Hyper vus dans les projets précédents.

Ce projet intègre : routage dynamique, load balancing, circuit breaker, cache de réponses, et transformation de requêtes.

**Environnement** : Rust async, Axum, Tower, `hyper`, `reqwest` (client HTTP), `tower-http`  
**Architecture** : configuration YAML hot-reloadable, health checks actifs sur les upstreams

---

## Objectifs du projet

1. Implémenter le **routage dynamique** : table de routes YAML → upstream cible
2. Implémenter le **load balancing** round-robin et least-connections entre plusieurs instances
3. Implémenter le **circuit breaker** : ouvre après N erreurs consécutives, demi-ouvre après timeout
4. Implémenter le **cache de réponses** : cache GET en mémoire (TTL par route, `Cache-Control` respecté)
5. Implémenter la **transformation de requêtes** : ajout/suppression de headers, réécriture de path
6. Implémenter le **hot-reload de configuration** : sans redémarrage, via `SIGHUP` ou endpoint `/admin/reload`

---

## Spécifications techniques

### Structure du projet

```
api-gateway/
├── Cargo.toml
├── config/
│   └── gateway.yaml            ← configuration des routes et upstreams
├── src/
│   ├── main.rs
│   ├── config/
│   │   ├── mod.rs
│   │   ├── model.rs            ← GatewayConfig, Route, Upstream deserialisés
│   │   └── watcher.rs          ← hot-reload via notify
│   ├── proxy/
│   │   ├── mod.rs
│   │   ├── handler.rs          ← handler Axum principal (catch-all)
│   │   ├── client.rs           ← pool de connexions HTTP vers upstreams
│   │   ├── transformer.rs      ← transformation requêtes/réponses
│   │   └── cache.rs            ← cache GET en mémoire
│   ├── balancer/
│   │   ├── mod.rs
│   │   ├── round_robin.rs
│   │   └── least_conn.rs
│   ├── circuit_breaker/
│   │   └── mod.rs              ← state machine Closed/Open/HalfOpen
│   ├── health/
│   │   └── checker.rs          ← health checks actifs sur les upstreams
│   └── admin/
│       └── routes.rs           ← /admin/config, /admin/upstreams, /admin/reload
└── tests/
    └── gateway_integration.rs
```

### Configuration YAML

```yaml
# config/gateway.yaml
gateway:
  port: 8080
  admin_port: 9090

upstreams:
  products-api:
    instances:
      - url: "http://localhost:3001"
        weight: 2
      - url: "http://localhost:3002"
        weight: 1
    health_check:
      path: "/health"
      interval_secs: 10
      timeout_ms: 2000
    circuit_breaker:
      failure_threshold: 5
      recovery_timeout_secs: 30

  users-api:
    instances:
      - url: "http://localhost:3003"
    health_check:
      path: "/health/ready"
      interval_secs: 15

routes:
  - path: "/api/products"
    methods: ["GET", "POST"]
    upstream: products-api
    strip_prefix: false
    cache:
      enabled: true
      ttl_secs: 30
      methods: ["GET"]
    headers:
      add:
        X-Gateway-Version: "1.0"
        X-Forwarded-By: "rust-gateway"
      remove:
        - Authorization        # ne pas forwarder le JWT vers ce service

  - path: "/api/users"
    methods: ["GET", "POST", "PUT", "DELETE"]
    upstream: users-api
    rewrite:
      from: "^/api/users"
      to: "/v2/users"
    auth_required: true        # valider le JWT au niveau gateway
```

### Circuit breaker state machine

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    Closed,                    // normal — laisse passer
    Open { until: Instant },   // bloqué — répond 503 directement
    HalfOpen,                  // test — laisse passer 1 requête
}

pub struct CircuitBreaker {
    state: Arc<RwLock<CircuitState>>,
    failure_count: Arc<AtomicU32>,
    config: CircuitBreakerConfig,
}

impl CircuitBreaker {
    /// Tenter d'exécuter une requête. Retourne Err(CircuitOpen) si le circuit est ouvert.
    pub async fn call<F, T, E>(&self, f: F) -> Result<T, GatewayError>
    where
        F: Future<Output = Result<T, E>>,
        E: std::error::Error,
    {
        match self.state() {
            CircuitState::Open { until } if Instant::now() < until => {
                return Err(GatewayError::CircuitOpen);
            }
            CircuitState::Open { .. } => self.transition(CircuitState::HalfOpen),
            _ => {}
        }

        match f.await {
            Ok(result) => {
                self.on_success();
                Ok(result)
            }
            Err(e) => {
                self.on_failure();
                Err(GatewayError::Upstream(e.to_string()))
            }
        }
    }

    fn on_failure(&self) {
        let count = self.failure_count.fetch_add(1, Ordering::SeqCst) + 1;
        if count >= self.config.failure_threshold {
            self.transition(CircuitState::Open {
                until: Instant::now() + Duration::from_secs(self.config.recovery_timeout_secs),
            });
        }
    }

    fn on_success(&self) {
        self.failure_count.store(0, Ordering::SeqCst);
        self.transition(CircuitState::Closed);
    }
}
```

### Load balancer least-connections

```rust
pub struct LeastConnections {
    instances: Vec<UpstreamInstance>,
    counters: Vec<Arc<AtomicU32>>,   // connexions actives par instance
}

impl LoadBalancer for LeastConnections {
    fn select(&self) -> Option<(&UpstreamInstance, Arc<AtomicU32>)> {
        // Choisir l'instance avec le moins de connexions actives (parmi les instances healthy)
        self.instances.iter()
            .zip(self.counters.iter())
            .filter(|(inst, _)| inst.is_healthy())
            .min_by_key(|(_, counter)| counter.load(Ordering::Relaxed))
            .map(|(inst, counter)| {
                counter.fetch_add(1, Ordering::SeqCst);
                (inst, counter.clone())
            })
    }
}

// RAII guard : décrémente le compteur à la fin de la requête
pub struct ConnectionGuard(Arc<AtomicU32>);
impl Drop for ConnectionGuard {
    fn drop(&mut self) { self.0.fetch_sub(1, Ordering::SeqCst); }
}
```

### Cache de réponses en mémoire

```rust
pub struct ResponseCache {
    store: DashMap<String, CachedEntry>,
    max_size: usize,
}

struct CachedEntry {
    body: Bytes,
    status: StatusCode,
    headers: HeaderMap,
    expires_at: Instant,
}

impl ResponseCache {
    /// Clé de cache = method + path + query string normalisé
    pub fn cache_key(req: &Request<Body>) -> String;

    pub fn get(&self, key: &str) -> Option<Response<Body>>;
    pub fn insert(&self, key: String, response: &Response<Body>, ttl: Duration);

    /// Éviction LRU quand max_size est atteint.
    pub fn evict_expired(&self);
}
```

### Handler proxy principal

```rust
/// Handler catch-all Axum : reçoit TOUTES les requêtes et les route vers l'upstream
pub async fn proxy_handler(
    State(state): State<GatewayState>,
    req: Request<Body>,
) -> Result<Response<Body>, GatewayError> {
    // 1. Trouver la route correspondante
    let route = state.router.match_route(req.method(), req.uri().path())
        .ok_or(GatewayError::NoRoute)?;

    // 2. Vérifier le cache (GET uniquement)
    if req.method() == Method::GET && route.cache.enabled {
        let key = ResponseCache::cache_key(&req);
        if let Some(cached) = state.cache.get(&key) {
            return Ok(cached);
        }
    }

    // 3. Sélectionner un upstream via load balancer
    let upstream = state.balancers[&route.upstream]
        .select()
        .ok_or(GatewayError::NoHealthyUpstream)?;

    // 4. Transformer la requête (headers, path rewrite)
    let outbound_req = state.transformer.transform_request(req, &route)?;

    // 5. Forwarder via circuit breaker
    let response = state.circuit_breakers[&route.upstream]
        .call(state.client.forward(upstream, outbound_req))
        .await?;

    // 6. Mettre en cache si applicable
    if req.method() == Method::GET && route.cache.enabled && response.status().is_success() {
        let key = ResponseCache::cache_key(&req_original);
        state.cache.insert(key, &response, Duration::from_secs(route.cache.ttl_secs));
    }

    Ok(response)
}
```

---

## Livrables attendus

- [ ] `config/model.rs` + `config/watcher.rs` : config YAML + hot-reload via `notify`
- [ ] `balancer/round_robin.rs` + `balancer/least_conn.rs` : deux stratégies, trait `LoadBalancer`
- [ ] `circuit_breaker/mod.rs` : state machine Closed/Open/HalfOpen avec compteurs atomiques
- [ ] `proxy/cache.rs` : cache GET avec TTL, éviction LRU, respect `Cache-Control: no-cache`
- [ ] `proxy/transformer.rs` : ajout/suppression headers, réécriture de path regex
- [ ] `health/checker.rs` : health checks actifs toutes les N secondes, mise à jour du statut upstream
- [ ] `admin/routes.rs` : `/admin/upstreams` (état de santé), `/admin/reload` (config), `/admin/circuit-breakers`
- [ ] Tests : route match, circuit breaker → 503 → recovery, cache hit/miss, load balancing round-robin

---

## Critères de qualité

- Le hot-reload ne coupe aucune connexion en cours (swap atomique de la config)
- Le circuit breaker ne bloque jamais le thread (état dans `Arc<RwLock<>>`)
- Les health checks tournent dans un background task Tokio séparé des requêtes
- `cargo clippy -- -D warnings` : zéro warning
- Sous 1000 req/sec, la latence ajoutée par le gateway est < 1 ms (mesurable avec `wrk`)

---

## Ressources

- Tower service docs : https://docs.rs/tower
- `hyper` reverse proxy example : https://github.com/hyperium/hyper/tree/master/examples
- Circuit breaker pattern : https://martinfowler.com/bliki/CircuitBreaker.html
- `notify` (file watcher) : https://docs.rs/notify
- Crate `pingora` (Cloudflare proxy en Rust, référence) : https://github.com/cloudflare/pingora
````
