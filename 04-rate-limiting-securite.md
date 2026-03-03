````markdown
# Projet 04 — Rate limiting, sécurité HTTP & CORS

## Contexte

Tu implémentes les couches de protection HTTP d'une API Axum en production.  
Une API exposée sur internet sans ces protections est compromise en quelques heures : brute-force sur `/auth/login`, scraping massif, injection de headers, clickjacking.

Ce projet couvre les middlewares Tower qui s'ajoutent à n'importe quelle API Axum existante, sans modifier les handlers.

**Environnement** : Rust async, Axum, Tower middleware, `tower_governor` ou rate limiter custom, Redis pour le sliding window distribué  
**Modèle** : middlewares composables en couches Tower — ordre des couches = ordre d'exécution

---

## Objectifs du projet

1. Implémenter un **rate limiter par IP** : sliding window 100 req/min, headers `X-RateLimit-*`
2. Implémenter un **rate limiter par user** : plus permissif que par IP (500 req/min) pour les authentifiés
3. Implémenter les **security headers HTTP** : CSP, HSTS, X-Frame-Options, X-Content-Type-Options
4. Implémenter la **politique CORS** configurable par environnement (dev : permissif, prod : liste blanche)
5. Implémenter la **protection brute-force login** : lockout progressif après 5 échecs en 15 min
6. Implémenter le **request ID** : `X-Request-Id` sur toutes les réponses pour le tracing distribué

---

## Spécifications techniques

### Structure du projet

```
rate-limiting-securite/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── app.rs                    ← router avec couches middleware
│   ├── middleware/
│   │   ├── mod.rs
│   │   ├── rate_limiter.rs       ← sliding window IP + user
│   │   ├── security_headers.rs  ← Tower layer headers sécurité
│   │   ├── cors.rs               ← politique CORS configurable
│   │   ├── request_id.rs        ← X-Request-Id
│   │   └── brute_force.rs       ← lockout progressif
│   ├── store/
│   │   ├── memory.rs             ← rate limit en mémoire (dev)
│   │   └── redis.rs              ← rate limit Redis (prod, distribué)
│   └── config.rs
└── tests/
    └── middleware_tests.rs
```

### Ordre des couches Tower (critique)

```rust
pub fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/auth/login",  post(login_handler))
        .route("/api/data",    get(data_handler))
        // Couches appliquées de l'extérieure vers l'intérieure (ordre inverse d'exécution)
        .layer(
            ServiceBuilder::new()
                // 1. Request ID (le plus externe — premier à s'exécuter)
                .layer(RequestIdLayer::new())
                // 2. Tracing (contextualise les logs avec le request ID)
                .layer(TraceLayer::new_for_http()
                    .make_span_with(|req: &Request<_>| {
                        let req_id = req.headers()
                            .get("x-request-id")
                            .and_then(|v| v.to_str().ok())
                            .unwrap_or("unknown");
                        tracing::info_span!("http_request", request_id = req_id)
                    })
                )
                // 3. Security headers
                .layer(SecurityHeadersLayer::new())
                // 4. CORS
                .layer(CorsLayer::new().allow_origin(state.config.allowed_origins.clone()))
                // 5. Rate limit par IP (le plus interne avant les routes)
                .layer(IpRateLimitLayer::new(state.rate_store.clone()))
        )
        .with_state(state)
}
```

### Rate limiter sliding window

```rust
/// Algorithme sliding window : plus précis qu'un compteur fixe (pas de burst en bord de fenêtre).
pub struct SlidingWindowLimiter {
    store: Arc<dyn RateLimitStore>,
    window_secs: u64,
    max_requests: u64,
}

impl SlidingWindowLimiter {
    /// Retourne Ok(remaining) ou Err(RetryAfter).
    pub async fn check(&self, key: &str) -> Result<RateLimitInfo, RateLimitExceeded>;
}

#[derive(Debug)]
pub struct RateLimitInfo {
    pub remaining: u64,
    pub reset_at: u64,      // UNIX timestamp
    pub limit: u64,
}

#[derive(Debug)]
pub struct RateLimitExceeded {
    pub retry_after_secs: u64,
}

// Headers retournés sur chaque réponse :
// X-RateLimit-Limit: 100
// X-RateLimit-Remaining: 87
// X-RateLimit-Reset: 1740000000
// Retry-After: 42  (seulement sur 429)
```

### Implémentation Redis (sliding window lua script)

```rust
// Script Lua atomique pour le sliding window côté Redis
const SLIDING_WINDOW_SCRIPT: &str = r#"
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

-- Supprimer les entrées hors fenêtre
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- Compter les requêtes dans la fenêtre
local count = redis.call('ZCARD', key)

if count < limit then
    -- Ajouter la requête courante
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window + 1)
    return {1, limit - count - 1}
else
    return {0, 0}
end
"#;
```

### Security headers

```rust
pub struct SecurityHeadersLayer;

impl<S> Layer<S> for SecurityHeadersLayer {
    type Service = SecurityHeadersMiddleware<S>;
    fn layer(&self, inner: S) -> Self::Service {
        SecurityHeadersMiddleware { inner }
    }
}

// Headers ajoutés sur toutes les réponses :
const SECURITY_HEADERS: &[(&str, &str)] = &[
    ("X-Content-Type-Options",  "nosniff"),
    ("X-Frame-Options",         "DENY"),
    ("X-XSS-Protection",        "1; mode=block"),
    ("Referrer-Policy",         "strict-origin-when-cross-origin"),
    ("Permissions-Policy",      "geolocation=(), microphone=(), camera=()"),
    (
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:"
    ),
    // HSTS uniquement en HTTPS (configurable)
    // ("Strict-Transport-Security", "max-age=31536000; includeSubDomains"),
];
```

### Lockout brute-force login

```rust
pub struct BruteForceProtection {
    store: Arc<dyn RateLimitStore>,
}

impl BruteForceProtection {
    /// Enregistre un échec d'authentification. Retourne le statut de lockout.
    pub async fn record_failure(&self, identifier: &str) -> LockoutStatus;

    /// Vérifie si l'identifiant est locké avant de tenter l'auth.
    pub async fn check(&self, identifier: &str) -> Result<(), LockoutStatus>;

    /// Réinitialise le compteur après un succès.
    pub async fn record_success(&self, identifier: &str);
}

#[derive(Debug)]
pub enum LockoutStatus {
    Ok { failures: u32 },
    Warning { failures: u32, remaining_attempts: u32 },
    Locked { retry_after_secs: u64 },
}

// Stratégie de backoff exponentiel :
// 1-4 échecs  → rien
// 5 échecs    → lockout 1 minute
// 6 échecs    → lockout 5 minutes
// 7+ échecs   → lockout 15 minutes, alerte
```

---

## Livrables attendus

- [ ] `middleware/rate_limiter.rs` : sliding window IP + user, headers `X-RateLimit-*`
- [ ] `store/memory.rs` : backend mémoire (DashMap + timestamps)
- [ ] `store/redis.rs` : backend Redis avec script Lua atomique
- [ ] `middleware/security_headers.rs` : Tower layer headers sécurité
- [ ] `middleware/cors.rs` : CORS configurable dev/prod
- [ ] `middleware/brute_force.rs` : lockout progressif avec backoff exponentiel
- [ ] `middleware/request_id.rs` : génération UUID v4 + propagation dans le span tracing
- [ ] Tests : rate limit déclenché à N+1, lockout après 5 échecs, headers présents sur toutes les réponses

---

## Critères de qualité

- Le rate limiter ne peut pas être contourné en parallélisant des requêtes (atomicité Redis)
- CORS en prod : seules les origines listées en config sont acceptées (pas de wildcard)
- Le lockout brute-force s'applique à l'email, pas seulement à l'IP (évite le contournement par VPN)
- `cargo clippy -- -D warnings` : zéro warning
- La couche middleware ne doit pas paniquer si Redis est indisponible — fallback sur mémoire locale

---

## Ressources

- Tower middleware guide : https://docs.rs/tower/latest/tower/
- `tower_governor` (rate limiting) : https://docs.rs/tower_governor
- OWASP rate limiting : https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks
- Security headers reference : https://securityheaders.com
- Redis sorted sets sliding window : https://redis.io/docs/data-types/sorted-sets/
````
