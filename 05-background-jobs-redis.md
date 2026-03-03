````markdown
# Projet 05 — Background jobs, queues Redis & retries

## Contexte

Tu implémentes un système de traitement de tâches asynchrones pour une API Axum.  
Certaines opérations ne peuvent pas attendre une réponse HTTP : envoi d'emails, génération de rapports PDF, appels à des API tierces lentes, traitement d'images.  
En production, on les délègue à un worker séparé via une **queue de messages**.

Ce projet implémente un système de jobs complet avec Redis comme backend : enqueue depuis l'API, traitement par des workers Tokio, retries avec backoff exponentiel, dead-letter queue, et dashboard de monitoring.

**Environnement** : Rust async, Axum, Redis (`deadpool-redis`), Tokio tasks  
**Patterns** : at-least-once delivery, idempotency keys, job deduplication

---

## Objectifs du projet

1. Implémenter une **queue Redis** avec enqueue/dequeue atomique via `LPUSH`/`BRPOP`
2. Implémenter un **worker pool** Tokio configurable (N workers concurrents)
3. Implémenter les **retries avec backoff exponentiel** + jitter (évite les thundering herds)
4. Implémenter la **dead-letter queue** (DLQ) : jobs échoués après max_retries
5. Implémenter les **idempotency keys** : garantir qu'un même job ne s'exécute pas deux fois
6. Implémenter un **endpoint de monitoring** : stats queue (pending, processing, failed, done)

---

## Spécifications techniques

### Structure du projet

```
background-jobs-redis/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── app.rs
│   ├── jobs/
│   │   ├── mod.rs
│   │   ├── queue.rs            ← JobQueue : enqueue, dequeue
│   │   ├── worker.rs           ← WorkerPool : N workers Tokio
│   │   ├── registry.rs         ← JobRegistry : routing type → handler
│   │   ├── retry.rs            ← politique de retry + backoff
│   │   └── dlq.rs              ← dead-letter queue
│   ├── handlers/
│   │   ├── email.rs            ← handler job "SendEmail"
│   │   ├── report.rs           ← handler job "GenerateReport"
│   │   └── webhook.rs          ← handler job "DeliverWebhook"
│   ├── routes/
│   │   ├── jobs.rs             ← POST /jobs, GET /jobs/:id
│   │   └── monitoring.rs       ← GET /jobs/stats
│   └── db.rs
└── tests/
    └── queue_integration.rs
```

### Structure d'un job

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Job {
    pub id: Uuid,
    pub idempotency_key: Option<String>,  // prévient la duplication
    pub job_type: String,                  // "send_email", "generate_report", ...
    pub payload: serde_json::Value,
    pub priority: Priority,
    pub status: JobStatus,
    pub attempts: u32,
    pub max_attempts: u32,
    pub last_error: Option<String>,
    pub scheduled_at: DateTime<Utc>,      // pour les jobs différés
    pub created_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum JobStatus {
    Pending, Processing, Completed, Failed, DeadLettered,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, PartialOrd, Ord)]
pub enum Priority { Low = 0, Normal = 1, High = 2, Critical = 3 }
```

### Queue Redis

```rust
pub struct JobQueue {
    redis: deadpool_redis::Pool,
    queue_name: String,
    processing_name: String,   // liste "processing" (jobs en cours)
    dlq_name: String,
}

impl JobQueue {
    /// Enqueue atomique avec déduplication par idempotency_key.
    pub async fn enqueue(&self, job: &Job) -> Result<EnqueueResult, QueueError>;

    /// Dequeue bloquant (BRPOP + LPUSH vers processing). Timeout 5 s.
    pub async fn dequeue(&self, timeout_secs: f64) -> Result<Option<Job>, QueueError>;

    /// Marquer un job comme terminé (retirer de processing).
    pub async fn ack(&self, job_id: Uuid) -> Result<(), QueueError>;

    /// Marquer un job comme échoué → retry ou DLQ.
    pub async fn nack(&self, job: Job, error: &str) -> Result<NackResult, QueueError>;

    pub async fn stats(&self) -> Result<QueueStats, QueueError>;
}

#[derive(Debug)]
pub enum EnqueueResult {
    Queued(Uuid),
    Duplicate,   // idempotency_key déjà vu
}

#[derive(Debug)]
pub enum NackResult {
    Retrying { attempt: u32, retry_at: DateTime<Utc> },
    DeadLettered,
}
```

### Retry avec backoff exponentiel + jitter

```rust
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub initial_delay_secs: f64,   // ex : 5
    pub multiplier: f64,           // ex : 2.0
    pub max_delay_secs: f64,       // ex : 3600 (1 heure)
    pub jitter_factor: f64,        // ex : 0.25 (±25%)
}

impl RetryPolicy {
    /// Calcule le délai avant la prochaine tentative.
    /// delay = min(initial * multiplier^(attempt-1), max) * (1 ± jitter)
    pub fn delay_for(&self, attempt: u32) -> Duration {
        let base = self.initial_delay_secs * self.multiplier.powi(attempt as i32 - 1);
        let capped = base.min(self.max_delay_secs);
        let jitter = capped * self.jitter_factor * (rand::random::<f64>() * 2.0 - 1.0);
        Duration::from_secs_f64((capped + jitter).max(0.0))
    }

    pub fn should_retry(&self, attempt: u32) -> bool {
        attempt < self.max_attempts
    }
}
```

### Worker pool

```rust
pub struct WorkerPool {
    queue: Arc<JobQueue>,
    registry: Arc<JobRegistry>,
    config: WorkerConfig,
}

#[derive(Clone)]
pub struct WorkerConfig {
    pub concurrency: usize,      // ex: 10 workers
    pub shutdown_timeout: Duration,
    pub retry_policy: RetryPolicy,
}

impl WorkerPool {
    pub async fn run(self, shutdown: CancellationToken) {
        let mut handles = Vec::new();
        for _ in 0..self.config.concurrency {
            let worker = Worker::new(self.queue.clone(), self.registry.clone(), self.config.clone());
            handles.push(tokio::spawn(worker.run(shutdown.clone())));
        }
        // Graceful shutdown : attendre les workers en cours
        shutdown.cancelled().await;
        for handle in handles { let _ = handle.await; }
    }
}
```

### Registry des job handlers

```rust
pub type JobResult = Result<(), Box<dyn std::error::Error + Send + Sync>>;
pub type JobHandler = Box<dyn Fn(serde_json::Value) -> BoxFuture<'static, JobResult> + Send + Sync>;

pub struct JobRegistry {
    handlers: HashMap<String, Arc<JobHandler>>,
}

impl JobRegistry {
    pub fn register<H, F>(&mut self, job_type: &str, handler: H)
    where
        H: Fn(serde_json::Value) -> F + Send + Sync + 'static,
        F: Future<Output = JobResult> + Send + 'static;

    pub fn dispatch(&self, job: &Job) -> Option<Arc<JobHandler>>;
}

// Enregistrement dans main
registry.register("send_email",      handlers::email::handle);
registry.register("generate_report", handlers::report::handle);
registry.register("deliver_webhook", handlers::webhook::handle);
```

### Stats de monitoring

```rust
#[derive(Debug, Serialize)]
pub struct QueueStats {
    pub pending: i64,
    pub processing: i64,
    pub dead_lettered: i64,
    pub completed_last_hour: i64,
    pub failed_last_hour: i64,
    pub avg_processing_time_ms: f64,
}
```

---

## Livrables attendus

- [ ] `jobs/queue.rs` : enqueue atomique (MULTI/EXEC Redis), dequeue BRPOP, ack/nack
- [ ] `jobs/retry.rs` : RetryPolicy avec backoff exponentiel + jitter, tests mathématiques
- [ ] `jobs/worker.rs` : WorkerPool avec concurrency configurable + graceful shutdown
- [ ] `jobs/registry.rs` : dispatch par job_type, handler async dynamique
- [ ] `jobs/dlq.rs` : promotion vers DLQ + endpoint de re-queue manuel
- [ ] 3 handlers exemples : email (simulé), report (sleep 500ms), webhook (reqwest)
- [ ] `routes/monitoring.rs` : `GET /jobs/stats`, `GET /jobs/:id`, `POST /jobs/:id/retry`
- [ ] Tests : enqueue → process → ack, retry après erreur, DLQ après max_attempts, déduplication

---

## Critères de qualité

- Le `dequeue` + `ack` est atomique : pas de job perdu si le worker crash (pattern reliable queue)
- Graceful shutdown : les jobs en cours se terminent avant l'arrêt (pas `kill -9`)
- Les idempotency keys sont stockées avec TTL approprié (ex: 24h)
- `cargo clippy -- -D warnings` : zéro warning
- Chaque job handler est indépendant et peut être testé isolément

---

## Ressources

- Reliable queue pattern Redis : https://redis.io/docs/manual/patterns/reliable-queue/
- `deadpool-redis` : https://docs.rs/deadpool-redis
- `tokio-util` CancellationToken : https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html
- Exponential backoff + jitter : https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
````
