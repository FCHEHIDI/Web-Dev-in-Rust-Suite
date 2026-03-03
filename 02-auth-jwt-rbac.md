````markdown
# Projet 02 — Authentification JWT + RBAC

## Contexte

Tu implémentes un système d'authentification et d'autorisation complet pour une API Axum.  
En production, l'authentification n'est pas juste "vérifier un token" : c'est gérer les refresh tokens, révoquer des sessions, implémenter des rôles granulaires, défendre contre le vol de token, et logguer les tentatives suspectes.

**Environnement** : Rust async, Axum, SQLx + PostgreSQL, `jsonwebtoken`, `argon2`  
**Securité** : tokens access (15 min) + refresh (7 jours) stockés en DB, rotation à chaque refresh  
**Autorisation** : RBAC — rôles `admin`, `editor`, `viewer` avec permissions par ressource

---

## Objectifs du projet

1. Implémenter le **hachage de mot de passe** avec Argon2id (standard actuel, résistant GPU)
2. Implémenter la **paire access/refresh token** : JWT RS256 (asymétrique, plus sûr qu'HS256)
3. Implémenter la **rotation des refresh tokens** : chaque refresh invalide l'ancien token
4. Implémenter un **extractor Axum** `AuthUser` qui valide le JWT et injecte le user dans le handler
5. Implémenter le **middleware RBAC** : macro ou guard `require_permission!(resource, action)`
6. Implémenter la **révocation de session** (logout, logout-all-devices) via table de tokens révoqués
7. Implémenter la **détection de réutilisation** de refresh token (signe de vol de session)

---

## Spécifications techniques

### Schéma de base de données

```sql
-- Migration 001
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,       -- argon2id
    role        TEXT NOT NULL DEFAULT 'viewer',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE refresh_tokens (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL UNIQUE,  -- hash du refresh token (jamais en clair)
    family      UUID NOT NULL,         -- famille pour détecter la réutilisation
    expires_at  TIMESTAMPTZ NOT NULL,
    revoked_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_family ON refresh_tokens(family);
```

### Endpoints d'authentification

```
POST /auth/register   → 201 { user_id, email }
POST /auth/login      → 200 { access_token, refresh_token, expires_in }
POST /auth/refresh    → 200 { access_token, refresh_token }   (rotation)
POST /auth/logout     → 204 (révoque le refresh token courant)
POST /auth/logout-all → 204 (révoque tous les refresh tokens du user)
GET  /auth/me         → 200 { id, email, role }               (requiert auth)
```

### Claims JWT

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct AccessTokenClaims {
    pub sub: Uuid,              // user_id
    pub email: String,
    pub role: Role,
    pub exp: i64,               // expiry UNIX timestamp
    pub iat: i64,               // issued at
    pub jti: Uuid,              // JWT ID unique (pour audit)
}

#[derive(Debug, Serialize, Deserialize)]
pub struct RefreshTokenClaims {
    pub sub: Uuid,              // user_id
    pub family: Uuid,           // famille de rotation
    pub jti: Uuid,              // ID unique stocké en DB (hashé)
    pub exp: i64,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "text")]
pub enum Role { Admin, Editor, Viewer }
```

### Extractor Axum `AuthUser`

```rust
/// Injecter dans n'importe quel handler pour exiger une authentification.
/// Retourne 401 si absent, 401 si expiré, 403 si révoqué.
#[derive(Debug, Clone)]
pub struct AuthUser {
    pub id: Uuid,
    pub email: String,
    pub role: Role,
}

#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync + AsRef<AppState>,
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        // 1. Extraire le header Authorization: Bearer <token>
        // 2. Valider la signature RS256 + expiry
        // 3. Retourner AuthUser depuis les claims
    }
}
```

### RBAC — permissions

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Permission {
    ProductRead, ProductWrite, ProductDelete,
    UserRead, UserWrite, UserDelete,
}

impl Role {
    pub fn has_permission(&self, perm: Permission) -> bool {
        match self {
            Role::Admin  => true,
            Role::Editor => matches!(perm,
                Permission::ProductRead | Permission::ProductWrite |
                Permission::UserRead),
            Role::Viewer => matches!(perm, Permission::ProductRead),
        }
    }
}

/// Guard à appeler en début de handler.
pub fn require(user: &AuthUser, perm: Permission) -> Result<(), AppError> {
    if user.role.has_permission(perm) {
        Ok(())
    } else {
        Err(AppError::Forbidden("insufficient permissions".into()))
    }
}
```

### Rotation des refresh tokens (protection vol de session)

```rust
pub async fn refresh_tokens(
    state: &AppState,
    refresh_token_raw: &str,
) -> Result<TokenPair, AppError> {
    // 1. Valider le JWT refresh
    let claims = validate_refresh_jwt(refresh_token_raw, &state.config.jwt_public_key)?;

    // 2. Chercher le token en DB par hash(token_raw)
    let stored = repo::find_refresh_token(&state.db, claims.jti).await?;

    // 3. Si déjà révoqué → RÉUTILISATION DÉTECTÉE
    //    Révoquer toute la famille (signe de vol de session)
    if stored.revoked_at.is_some() {
        tracing::warn!(family = %claims.family, "refresh token reuse detected, revoking family");
        repo::revoke_token_family(&state.db, claims.family).await?;
        return Err(AppError::Unauthorized("session compromised".into()));
    }

    // 4. Révoquer l'ancien, en émettre un nouveau dans la même famille
    repo::revoke_refresh_token(&state.db, stored.id).await?;
    let new_pair = issue_token_pair(&state.db, &state.config, claims.sub).await?;
    Ok(new_pair)
}
```

---

## Livrables attendus

- [ ] `auth/password.rs` : hachage Argon2id + vérification (paramètres mémoire 64 MB, 3 itérations)
- [ ] `auth/jwt.rs` : génération + validation RS256 access/refresh tokens
- [ ] `auth/extractor.rs` : `AuthUser` extractor Axum complet
- [ ] `auth/rbac.rs` : enum `Permission`, `Role::has_permission`, `require()`
- [ ] `routes/auth.rs` : 6 endpoints register/login/refresh/logout/logout-all/me
- [ ] Détection de réutilisation de refresh token avec révocation de famille
- [ ] Tests : login → refresh → logout, réutilisation détectée, rôle insuffisant → 403
- [ ] Test de charge : 1000 validations JWT/sec sans appel DB (claims auto-suffisants)

---

## Critères de qualité

- Le refresh token n'est **jamais stocké en clair** en base (SHA256 du token)
- Les clés RS256 sont chargées depuis fichiers PEM, pas depuis des constantes
- `argon2` configuré avec des paramètres résistants : m=65536, t=3, p=1 minimum
- Tous les endpoints auth ont des tests de cas limites (token expiré, malformé, révoqué)
- `cargo clippy -- -D warnings` : zéro warning
- Aucun secret (`jwt_private_key`, `database_url`) dans le code source

---

## Ressources

- Crate `jsonwebtoken` : https://docs.rs/jsonwebtoken
- Crate `argon2` : https://docs.rs/argon2
- OWASP JWT cheat sheet : https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
- Refresh token rotation : https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation
- Code ALVR auth (référence pattern Rust) : `alvr/server/src/connection.rs` (handshake session)
````
