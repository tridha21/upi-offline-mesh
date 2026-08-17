# UPI Offline Mesh

A Spring Boot demo of **offline-first UPI payments** routed through a phone-to-phone mesh network.

You're in a basement with zero signal. You send a friend ₹500. Your phone encrypts the payment and broadcasts it to nearby phones over "Bluetooth." The packet hops device to device until *some* phone walks outside, picks up 4G, and quietly uploads it to this backend — which decrypts it, checks it hasn't been seen before, and settles it exactly once.

This repo is the **server side** of that system, plus a built-in simulator of the mesh, so you can run the entire flow on one laptop — no real Bluetooth hardware, no other devices needed.

---

## What it demonstrates

- **End-to-end encryption across untrusted hops** — every intermediate device carries the payment but can't read or tamper with it (RSA-OAEP wraps an AES-256-GCM session key).
- **Exactly-once settlement under concurrency** — if the same packet reaches the backend through multiple bridge devices at once, only one settles. The rest are dropped as duplicates via an idempotency check on the ciphertext hash.
- **Tamper and replay rejection** — a modified or re-sent packet is rejected before it ever touches the ledger.

All three are visible live in the dashboard.

---

## Live Demo

The application is publicly deployed and available to try online:

**https://upi-offline-mesh-v0tz.onrender.com/**

The live deployment provides the complete interactive command center, including:

- Live mesh network simulation
- Encrypted payment packet creation
- Virtual device-to-device gossip
- Bridge upload simulation
- Exactly-once settlement
- Idempotency and duplicate detection
- Account balances
- Transaction ledger
- Live activity stream

### Try the demo

1. Open the live demo.
2. Select a sender and receiver.
3. Enter an amount and demo PIN.
4. Click **Inject into mesh**.
5. Run a **gossip round** to move the packet through the simulated devices.
6. Click **Upload from bridges** to send the packet to the backend.
7. Watch the account balance and transaction ledger update.

The application automatically refreshes the network state every few seconds.

> **Note:** This is an educational proof of concept. It does not process real money and is not connected to NPCI, UPI, banks, or real payment providers.

---

## Running it

**Prerequisite:** JDK 17+ on your PATH (`java -version` to check). Nothing else — no database or Redis to install, the Maven wrapper handles the build.

```bash
# Windows
mvnw.cmd spring-boot:run

# Mac / Linux
./mvnw spring-boot:run
```

First run downloads dependencies (a couple of minutes); after that, startup takes a few seconds. Once you see `Started UpiMeshApplication`, open:

**http://localhost:8080**

Stop the server with `Ctrl+C`.

Run the tests:

```bash
# Windows
mvnw.cmd test

# Mac / Linux
./mvnw test
```

---

## Using the dashboard

The dashboard walks through the pipeline in order:

1. **Compose payment** — pick a sender, receiver, amount, and PIN, then inject it into the mesh. This simulates the sender's phone building and encrypting the packet.
2. **Run a gossip round** — packets hop between virtual devices, the way phones would pass a packet around as people walk past each other.
3. **Bridge upload** — the device(s) with simulated internet access upload everything they're holding to the backend in parallel, which is what actually exercises the idempotency logic.
4. **Reset** — clears the mesh and the idempotency cache to start over.

Balances and the transaction ledger update live as packets settle, and the activity log shows exactly what happened at each step.

To see the duplicate-delivery case properly (multiple bridges delivering the same packet at once), run:

```bash
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

That test fires three threads at the ingest pipeline simultaneously with the same packet and asserts exactly one settles.

---

## How a payment moves through the system

```text
Sender phone (offline)
  → builds PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }
  → encrypts with the server's RSA public key (hybrid RSA + AES-GCM)
  → wraps it in a MeshPacket { packetId, ttl, ciphertext }
  → hands it to a nearby device

Mesh gossip (Bluetooth-style, simulated)
  device → device → device → bridge device (gets 4G)

Bridge device
  → POST /api/bridge/ingest

Backend
  1. hash the ciphertext (SHA-256)
  2. try to claim that hash in the idempotency cache — first claim wins
  3. decrypt with the server's RSA private key
  4. check freshness (rejects anything older than 24h)
  5. debit sender / credit receiver / write a ledger row, all in one transaction
```

---

## Project layout

```text
src/main/java/com/demo/upimesh/
├── model/        Account, Transaction, MeshPacket, PaymentInstruction (JPA + wire types)
├── crypto/       RSA keypair generation + hybrid RSA/AES encrypt-decrypt
├── service/      DemoService, MeshSimulatorService, IdempotencyService,
│                 SettlementService, BridgeIngestionService (the core pipeline)
├── controller/   ApiController (REST endpoints), DashboardController (serves the UI)
└── config/       Scheduling config for cache eviction

src/main/resources/templates/dashboard.html   the interactive demo UI
src/test/java/.../IdempotencyConcurrencyTest.java   the concurrency + tamper tests
```

---

## API reference

| Method | Path | What it does |
|---|---|---|
| GET | `/` | Dashboard UI |
| GET | `/api/server-key` | Server's RSA public key |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Last 20 transactions |
| GET | `/api/mesh/state` | Current state of every virtual device |
| POST | `/api/demo/send` | Simulate a sender phone: encrypt + inject a packet |
| POST | `/api/mesh/gossip` | Run one round of gossip |
| POST | `/api/mesh/flush` | Bridges with internet upload to the backend (parallel) |
| POST | `/api/mesh/reset` | Clear the mesh + idempotency cache |
| POST | `/api/bridge/ingest` | **The production-facing endpoint** — real bridge devices would POST here |
| GET | `/h2-console` | Browse the in-memory database (`jdbc:h2:mem:upimesh`, user `sa`, no password) |

### `/api/bridge/ingest`

The endpoint expects a `MeshPacket` JSON body plus:

```text
X-Bridge-Node-Id: bridge-01
X-Hop-Count: 3
```

Example response:

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible outcomes include:

```text
SETTLED
DUPLICATE_DROPPED
INVALID
```

---

## What's simulated vs. what's real

| Demo | Production equivalent |
|---|---|
| H2 in-memory DB | PostgreSQL / MySQL |
| `ConcurrentHashMap` idempotency cache | Redis (`SET NX EX`) or another distributed idempotency store |
| RSA keypair regenerated on startup | Private key stored in an HSM / KMS |
| `MeshSimulatorService` software gossip | Real BLE / Wi-Fi Direct between phones |
| No authentication on `/api/bridge/ingest` | Mutual TLS or signed bridge certificates |
| Virtual devices | Physical Android devices |
| Simulated internet availability | Real connectivity changes |

---

## Honest limitations of the concept

These aren't bugs — they're inherent to "no internet, anywhere in the chain":

- **The receiver can't verify funds offline.** A packet shown as "sent" is really an IOU until it settles. If the sender's balance is insufficient by the time it reaches the backend, the transaction is rejected and the receiver has no recourse. Real offline payment products solve this through pre-funded balances and controlled offline limits.
- **A malicious sender could double-spend** by sending conflicting packets into two different offline pockets — whichever reaches the backend first wins.
- **Real background Bluetooth is hard.** Android throttles background BLE heavily, while iOS imposes additional restrictions on background Bluetooth behavior. This demo sidesteps that complexity by simulating the mesh in software.
- **This is not real UPI integration.** The project does not connect to NPCI, banks, UPI switches, or real bank accounts. It models the distributed-system and security concepts around deferred settlement.
- **The server currently trusts the bridge transport.** Production deployment would require strong bridge authentication, authorization, replay controls, rate limiting, and transport security.

The most accurate description of the concept is **"mesh-routed deferred settlement"**, rather than "real-time offline payments." Deferred settlement is the core trade-off being demonstrated.

---

## Security notes

### Hybrid encryption

The payment payload uses:

```text
PaymentInstruction
       │
       ▼
   AES-256-GCM
       │
       ├── Ciphertext
       └── Authentication tag

AES session key
       │
       ▼
RSA-OAEP / SHA-256
       │
       ▼
Encrypted AES key
```

The server's RSA public key is used to protect the session key, while AES-GCM encrypts the actual payment data.

### Idempotency

Before settlement, the backend hashes the encrypted packet:

```text
ciphertext
    │
    ▼
SHA-256
    │
    ▼
idempotency claim
    │
    ├── first claim → continue
    └── existing    → DUPLICATE_DROPPED
```

This is what prevents multiple bridge uploads of the same packet from creating multiple settlements.

### Freshness / replay protection

The backend checks the payment timestamp and rejects packets outside the allowed freshness window. The current implementation accepts packets up to **24 hours old** and rejects packets that are more than **5 minutes in the future**.

---

## Testing

Run the full test suite:

```bash
# Windows
mvnw.cmd test

# Mac / Linux
./mvnw test
```

Run the concurrency scenario:

```bash
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

The concurrency test delivers the same packet from three bridge threads simultaneously and verifies that:

```text
3 concurrent deliveries
        │
        ▼
┌─────────────────────┐
│ Idempotency check   │
└─────────┬───────────┘
          │
          ├── Thread 1 → SETTLED
          ├── Thread 2 → DUPLICATE_DROPPED
          └── Thread 3 → DUPLICATE_DROPPED
```

The ledger should contain only one successful settlement.

---

## Troubleshooting

### `java: command not found`

Install JDK 17+ and verify:

```bash
java -version
```

On Windows with `winget`:

```powershell
winget install EclipseAdoptium.Temurin.17.JDK
```

### Port 8080 is already in use

Change the port in:

```text
src/main/resources/application.properties
```

For example:

```properties
server.port=8081
```

Then open:

```text
http://localhost:8081
```

### First run appears to hang

The Maven wrapper may be downloading Maven and project dependencies. Give it a couple of minutes on the first run.

### PowerShell won't run `mvnw.cmd`

Use:

```powershell
.\mvnw.cmd spring-boot:run
```

instead of:

```powershell
mvnw.cmd spring-boot:run
```

---

## Future improvements

Possible next steps for turning the demo into a more realistic system include:

- Replace H2 with PostgreSQL
- Replace the in-memory idempotency cache with Redis
- Add persistent key management using KMS/HSM
- Add bridge authentication and authorization
- Implement signed bridge certificates
- Build a real Android BLE transport
- Add device discovery and peer authentication
- Add packet signatures and stronger anti-replay mechanisms
- Add rate limiting and fraud detection
- Add observability, metrics, and distributed tracing
- Add failure recovery for partially completed mesh transfers

---

## License

This project is developed for **educational and demonstration purposes**.

You are welcome to explore, modify, and use the code for learning and personal projects. If you build upon this project, attribution to the original repository is appreciated.
