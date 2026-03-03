# RustWebDev Suite

Collection de projets **Rust HTTP/Web** à construire avec Claude Sonnet 4.6.  
Chaque fichier `.md` est un brief de projet autonome : contexte, objectifs, spécifications techniques, livrables attendus et critères de qualité.  
Axé sur les patterns de production réels : zéro `unwrap`, erreurs typées, observabilité, sécurité, scalabilité.

## Structure

| Fichier | Projet | Niveau | Repository |
|---|---|---|---|
| [01-api-rest-axum.md](01-api-rest-axum.md) | API REST production-ready avec Axum | Fondation | [FCHEHIDI/01-api-rest-axum](https://github.com/FCHEHIDI/01-api-rest-axum) |
| [02-auth-jwt-rbac.md](02-auth-jwt-rbac.md) | Authentification JWT + RBAC | Fondation | — |
| [03-websocket-temps-reel.md](03-websocket-temps-reel.md) | WebSocket temps réel & pub/sub | Intermédiaire | — |
| [04-rate-limiting-securite.md](04-rate-limiting-securite.md) | Rate limiting, sécurité HTTP, CORS | Intermédiaire | — |
| [05-background-jobs-redis.md](05-background-jobs-redis.md) | Background jobs, queues Redis, retries | Intermédiaire | — |
| [06-observabilite-otel.md](06-observabilite-otel.md) | Observabilité : tracing, métriques, logs (OpenTelemetry) | Avancé | — |
| [07-grpc-tonic-streaming.md](07-grpc-tonic-streaming.md) | gRPC avec Tonic + server streaming | Avancé | — |
| [08-api-gateway-proxy.md](08-api-gateway-proxy.md) | API Gateway / reverse proxy service mesh | Expert | — |

## Stack commune

- **Framework HTTP** : `axum` (Tokio-native, compositional)
- **Base de données** : `sqlx` async (PostgreSQL), migrations
- **Sérialisation** : `serde`, `serde_json`
- **Erreurs** : `thiserror` (définition), `anyhow` (propagation)
- **Async** : `tokio` (runtime), `tower` (middleware)
- **Tests** : `axum-test` ou `reqwest`, `sqlx::test`
- **Qualité** : `clippy`, `cargo-audit`, `cargo-tarpaulin` (coverage)

## Lien avec les patterns ALVR (backend distribué)

Ces projets partagent avec ALVR les mêmes fondamentaux Rust async :

| Pattern | WebDev suite | ALVR |
|---|---|---|
| Async channels | `tokio::sync::mpsc` pour jobs | transport vidéo frames |
| State partagé | `Arc<RwLock<>>` API state | encoder config partagée |
| Middleware tower | auth, rate limit, tracing | — |
| Sérialisation bincode/JSON | protocoles HTTP | protocoles réseau |

## Comment utiliser ces briefs avec Claude Sonnet 4.6

1. Ouvre le fichier `.md` du projet choisi dans VS Code
2. Copie le contenu complet dans une nouvelle conversation avec Claude
3. Démarre la session avec : _"Je veux implémenter ce projet, commençons par l'arborescence et le Cargo.toml"_
4. Itère section par section en suivant les livrables définis

---
*Généré le 03/03/2026 — FCHEHIDI*