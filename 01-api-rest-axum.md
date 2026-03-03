````markdown
# Projet 01 — API REST production-ready avec Axum

## Contexte

Tu implémentes une API REST complète en Rust avec **Axum**, le framework HTTP de la stack Tokio.  
L'objectif n'est pas un CRUD basique : c'est une API structurée comme en production — gestion d'erreurs typées, validation des entrées, pagination, tests d'intégration avec vraie base de données, et documentation OpenAPI générée automatiquement.

**Domaine** : API de gestion de ressources (ex : catalogue de produits)  
**Environnement** : Rust async, Axum 0.7+, SQLx + PostgreSQL, Tokio  
**Base de données** : PostgreSQL via `sqlx` avec migrations versionnées

---

## Objectifs du projet

1. Structurer un projet Axum avec **séparation des couches** : routes, handlers, services, repository
2. Implémenter les **5 opérations CRUD** avec validation des payloads via `validator`
3. Implémenter la **pagination cursor-based** (plus performante que offset/limit en prod)
4. Implémenter une **gestion d'erreurs centralisée** : `AppError` → réponses HTTP cohérentes
5. Implémenter les **migrations SQLx** versionnées et rejouer dans les tests
6. Générer la **documentation OpenAPI** automatiquement via `utoipa`
7. Écrire des **tests d'intégration** avec `sqlx::test` (base de données réelle par test)

---

## Spécifications techniques

### Structure du projet

```
api-rest-axum/
├── Cargo.toml
├── migrations/
│   ├── 001_create_products.sql
│   └── 002_add_indexes.sql
├── src/
│   ├── main.rs
│   ├── app.rs                  ← router Axum + état partagé
│   ├── config.rs               ← config desde variables d'environnement
│   ├── error.rs                ← AppError + impl IntoResponse
│   ├── db.rs                   ← pool SQLx + helpers
│   ├── models/
│   │   └── product.rs          ← structs DB + serde
│   ├── dto/
│   │   ├── product_request.rs  ← CreateProductRequest, UpdateProductRequest
│   │   └── product_response.rs ← ProductResponse, PageResponse<T>
│   ├── repository/
│   │   └── product_repo.rs     ← queries SQLx
│   ├── services/
│   │   └── product_service.rs  ← logique métier
│   └── routes/
│       ├── mod.rs
│       └── products.rs         ← handlers HTTP
└── tests/
    └── products_integration.rs
```

### Gestion d'erreurs centralisée

```rust
// Le pattern central de toute API Axum pro : un enum d'erreurs avec IntoResponse
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Resource not found: {0}")]
    NotFound(String),

    #[error("Validation error: {0}")]
    Validation(String),

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error("Database error")]
    Database(#[from] sqlx::Error),

    #[error("Internal error")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, body) = match &self {
            AppError::NotFound(msg)    => (StatusCode::NOT_FOUND,       error_body("NOT_FOUND", msg)),
            AppError::Validation(msg)  => (StatusCode::UNPROCESSABLE_ENTITY, error_body("VALIDATION_ERROR", msg)),
            AppError::Conflict(msg)    => (StatusCode::CONFLICT,        error_body("CONFLICT", msg)),
            AppError::Database(e)      => {
                tracing::error!(error = %e, "database error");
                (StatusCode::INTERNAL_SERVER_ERROR, error_body("DB_ERROR", "internal error"))
            }
            AppError::Internal(e)      => {
                tracing::error!(error = ?e, "internal error");
                (StatusCode::INTERNAL_SERVER_ERROR, error_body("INTERNAL_ERROR", "internal error"))
            }
        };
        (status, Json(body)).into_response()
    }
}

#[derive(Serialize)]
struct ErrorBody { code: &'static str, message: String }
fn error_body(code: &'static str, msg: &str) -> ErrorBody {
    ErrorBody { code, message: msg.to_owned() }
}
```

### Pagination cursor-based

```rust
// Meilleure performance qu'OFFSET/LIMIT : pas de scan complet à chaque page
#[derive(Debug, Deserialize)]
pub struct CursorPaginationParams {
    pub after: Option<Uuid>,    // cursor = dernier ID vu
    pub limit: Option<u32>,     // défaut : 20, max : 100
}

#[derive(Debug, Serialize)]
pub struct PageResponse<T> {
    pub data: Vec<T>,
    pub next_cursor: Option<Uuid>,   // None = dernière page
    pub total_count: i64,
}

// Query SQLx correspondante
async fn list_products(
    pool: &PgPool,
    after: Option<Uuid>,
    limit: i64,
) -> Result<Vec<Product>, sqlx::Error> {
    sqlx::query_as!(
        Product,
        r#"SELECT * FROM products
           WHERE ($1::uuid IS NULL OR id > $1)
           ORDER BY id ASC
           LIMIT $2"#,
        after,
        limit
    )
    .fetch_all(pool)
    .await
}
```

### État partagé Axum

```rust
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub config: Arc<Config>,
}

// Enregistrement dans le router
pub fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/products",       get(list_products).post(create_product))
        .route("/products/:id",   get(get_product).put(update_product).delete(delete_product))
        .route("/health",         get(health_check))
        .route("/docs",           get(openapi_ui))
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

### Validation des entrées

```rust
#[derive(Debug, Deserialize, Validate, ToSchema)]
pub struct CreateProductRequest {
    #[validate(length(min = 1, max = 200))]
    pub name: String,

    #[validate(range(min = 0.01))]
    pub price: f64,

    #[validate(length(max = 1000))]
    pub description: Option<String>,

    #[validate(range(min = 0))]
    pub stock: i32,
}

// Extractor custom qui valide automatiquement
pub struct ValidatedJson<T>(pub T);

#[async_trait]
impl<T, S> FromRequest<S> for ValidatedJson<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|e| AppError::Validation(e.to_string()))?;
        value.validate().map_err(|e| AppError::Validation(e.to_string()))?;
        Ok(ValidatedJson(value))
    }
}
```

---

## Livrables attendus

- [ ] Router Axum complet avec les 5 opérations CRUD + `/health` + `/docs`
- [ ] `AppError` avec `IntoResponse` couvrant tous les cas (NotFound, Validation, Conflict, DB, Internal)
- [ ] Pagination cursor-based fonctionnelle et testée
- [ ] `ValidatedJson<T>` extractor réutilisable
- [ ] Migrations SQLx versionnées (table + index)
- [ ] Documentation OpenAPI générée via `utoipa` (accessible sur `/docs`)
- [ ] Tests d'intégration : CRUD complet, pagination, cas d'erreur (404, 422, 409)
- [ ] `GET /health` retourne `{"status":"ok","db":"ok","version":"x.y.z"}`

---

## Critères de qualité

- Zéro `unwrap()` hors `main` et tests
- Chaque handler retourne `Result<impl IntoResponse, AppError>` — jamais de status 500 silencieux
- Les tests d'intégration utilisent `#[sqlx::test]` avec base de données réelle (pas de mocks)
- `cargo clippy -- -D warnings` : zéro warning
- Le pool SQLx est configuré : `max_connections`, `acquire_timeout`, `idle_timeout`
- Les secrets (DATABASE_URL) sont lus depuis l'environnement, jamais hardcodés

---

## Ressources

- Documentation Axum : https://docs.rs/axum
- SQLx : https://docs.rs/sqlx
- utoipa (OpenAPI) : https://docs.rs/utoipa
- validator : https://docs.rs/validator
- Patterns Axum pro : https://github.com/tokio-rs/axum/tree/main/examples
````
