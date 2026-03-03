````markdown
# Projet 03 — WebSocket temps réel & pub/sub

## Contexte

Tu implémentes un système de communication temps réel pour une API Axum : notifications live, chat, dashboard de métriques mis à jour en push.  
En production, le WebSocket n'est pas seul : il s'appuie sur un **bus de messages** (ici `tokio::sync::broadcast`) pour diffuser les événements entre connexions, et sur des **rooms** pour limiter la portée des diffusions.

Ce projet couvre le pattern complet que l'on retrouve dans les dashboards SaaS, les systèmes de collaboration, et les plateformes de monitoring.

**Environnement** : Rust async, Axum WebSocket, Tokio broadcast, Redis pub/sub (optionnel scale-out)  
**Patterns** : rooms, heartbeat/ping-pong, reconnexion côté client, backpressure

---

## Objectifs du projet

1. Implémenter des **connexions WebSocket authentifiées** (le JWT est validé au handshake HTTP)
2. Implémenter un système de **rooms** : un client peut rejoindre/quitter des rooms, recevoir uniquement les messages de ses rooms
3. Implémenter le **broadcast** intra-room via `tokio::sync::broadcast`
4. Implémenter le **heartbeat** : ping/pong toutes les 30 s, déconnexion après 90 s sans réponse
5. Implémenter la **gestion de la backpressure** : le client lent ne bloque pas les autres
6. Implémenter un **store de présence** : liste des utilisateurs connectés par room

---

## Spécifications techniques

### Structure du projet

```
websocket-realtime/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── app.rs
│   ├── error.rs
│   ├── ws/
│   │   ├── mod.rs
│   │   ├── handler.rs          ← upgrade HTTP → WebSocket
│   │   ├── connection.rs       ← boucle read/write par connexion
│   │   ├── hub.rs              ← gestionnaire global des rooms
│   │   └── message.rs          ← types de messages JSON
│   ├── presence/
│   │   └── store.rs            ← qui est connecté dans quelle room
│   └── routes/
│       ├── ws.rs               ← GET /ws (upgrade)
│       └── rooms.rs            ← REST : POST /rooms, GET /rooms/:id/members
└── tests/
    └── ws_integration.rs
```

### Messages WebSocket (JSON)

```rust
/// Messages envoyés par le CLIENT vers le serveur
#[derive(Debug, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ClientMessage {
    JoinRoom  { room_id: String },
    LeaveRoom { room_id: String },
    Send      { room_id: String, content: String },
    Pong      { echo: u64 },
}

/// Messages envoyés par le SERVEUR vers le client
#[derive(Debug, Serialize, Clone)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ServerMessage {
    Welcome   { user_id: Uuid, server_time: i64 },
    Joined    { room_id: String, member_count: usize },
    Left      { room_id: String },
    Message   { room_id: String, from: Uuid, content: String, ts: i64 },
    Presence  { room_id: String, event: PresenceEvent, user_id: Uuid },
    Ping      { echo: u64 },
    Error     { code: String, message: String },
}

#[derive(Debug, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum PresenceEvent { Joined, Left }
```

### Hub — gestionnaire des rooms

```rust
pub struct Hub {
    // map room_id → canal broadcast
    rooms: DashMap<String, broadcast::Sender<ServerMessage>>,
    // compteur de membres par room
    presence: Arc<PresenceStore>,
}

impl Hub {
    pub fn new() -> Arc<Self>;

    /// Rejoindre une room : retourne un Receiver pour recevoir les messages.
    pub fn join_room(&self, room_id: &str, user_id: Uuid)
        -> broadcast::Receiver<ServerMessage>;

    /// Quitter une room.
    pub fn leave_room(&self, room_id: &str, user_id: Uuid);

    /// Diffuser un message dans une room.
    pub fn broadcast(&self, room_id: &str, msg: ServerMessage) -> Result<usize, HubError>;

    /// Supprimer les rooms vides (GC périodique).
    pub async fn cleanup_empty_rooms(&self);
}
```

### Boucle de connexion par client

```rust
pub async fn handle_connection(
    ws: WebSocket,
    user: AuthUser,
    hub: Arc<Hub>,
) {
    let (mut sender, mut receiver) = ws.split();
    let mut subscriptions: HashMap<String, broadcast::Receiver<ServerMessage>> = HashMap::new();
    let mut ping_interval = tokio::time::interval(Duration::from_secs(30));
    let mut last_pong = Instant::now();

    loop {
        tokio::select! {
            // Message entrant du client
            msg = receiver.next() => {
                match msg {
                    Some(Ok(Message::Text(text))) => handle_client_message(&text, &user, &hub, &mut subscriptions, &mut sender).await,
                    Some(Ok(Message::Close(_))) | None => break,
                    _ => {}
                }
            }

            // Message sortant depuis une room abonnée
            // (itère sur toutes les subscriptions actives)
            msg = recv_any_room(&mut subscriptions) => {
                if let Some(server_msg) = msg {
                    let _ = sender.send(Message::Text(serde_json::to_string(&server_msg).unwrap())).await;
                }
            }

            // Heartbeat
            _ = ping_interval.tick() => {
                if last_pong.elapsed() > Duration::from_secs(90) {
                    break; // timeout
                }
                let _ = sender.send(Message::Text(
                    serde_json::to_string(&ServerMessage::Ping { echo: epoch_ms() }).unwrap()
                )).await;
            }
        }
    }

    // Nettoyage : quitter toutes les rooms
    for (room_id, _) in &subscriptions {
        hub.leave_room(room_id, user.id);
    }
}
```

### Backpressure : canal broadcast avec lagged

```rust
// broadcast::channel avec capacité bornée — les clients lents reçoivent Lagged
const ROOM_CHANNEL_CAPACITY: usize = 128;

// Si un receiver est trop lent et rate des messages, il reçoit Err(RecvError::Lagged(n))
// → envoyer au client un message d'erreur spécial et le déconnecter proprement
match room_rx.recv().await {
    Ok(msg) => { /* envoyer */ }
    Err(broadcast::error::RecvError::Lagged(n)) => {
        let _ = sender.send(Message::Text(
            serde_json::to_string(&ServerMessage::Error {
                code: "LAGGED".into(),
                message: format!("{n} messages missed, reconnect recommended"),
            }).unwrap()
        )).await;
    }
    Err(broadcast::error::RecvError::Closed) => break,
}
```

---

## Livrables attendus

- [ ] `ws/hub.rs` : Hub avec DashMap rooms, join/leave/broadcast, cleanup
- [ ] `ws/connection.rs` : boucle tokio::select! read/write/broadcast/heartbeat
- [ ] `ws/message.rs` : ClientMessage et ServerMessage avec serde tag polymorphique
- [ ] `presence/store.rs` : set de membres par room (DashMap<String, HashSet<Uuid>>)
- [ ] `routes/ws.rs` : upgrade WebSocket avec validation JWT au handshake
- [ ] `routes/rooms.rs` : REST CRUD rooms + GET /rooms/:id/members
- [ ] Tests : connexion, rejoindre une room, broadcast, déconnexion propre, heartbeat timeout
- [ ] Test de charge : 500 connexions simultanées, broadcast dans une room → latence < 10 ms

---

## Critères de qualité

- Aucune connexion ne bloque les autres (boucle `select!` non-bloquante)
- Les rooms vides sont nettoyées (pas de fuite mémoire sur DashMap)
- La déconnexion (propre ou crash réseau) déclenche toujours le cleanup présence
- `cargo clippy -- -D warnings` : zéro warning
- Tests avec `tokio::test` et `axum-test` ou les helpers WebSocket de `tungstenite`

---

## Ressources

- Axum WebSocket example : https://github.com/tokio-rs/axum/tree/main/examples/websockets
- `tokio::sync::broadcast` : https://docs.rs/tokio/latest/tokio/sync/broadcast/
- `dashmap` (concurrent HashMap) : https://docs.rs/dashmap
- WebSocket RFC 6455 : ping/pong frames
````
