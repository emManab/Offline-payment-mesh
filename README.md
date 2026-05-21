# 📡 UPI Offline Mesh — Professional Demo Backend

[![Java](https://img.shields.io/badge/Java-25%20LTS-ED8936?style=flat-square&logo=openjdk)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0.0-6DB33F?style=flat-square&logo=spring)](https://spring.io/projects/spring-boot)
[![Maven](https://img.shields.io/badge/Maven-3.9.9-C71A36?style=flat-square&logo=apache-maven)](https://maven.apache.org/)

> A production-shaped **offline payment routing simulation**: encrypted packets move across a virtual mesh and settle exactly once when a bridge node regains internet connectivity.

---

## Executive Summary

This project demonstrates how an offline-first UPI-like flow can be implemented safely at backend level:

- ✅ **Confidentiality & integrity** over untrusted intermediaries (RSA-OAEP + AES-256-GCM)
- ✅ **Exactly-once settlement** under duplicate concurrent deliveries (idempotency claim on ciphertext hash)
- ✅ **Replay/tamper defense** before ledger mutation

It includes:

1. A Spring Boot backend pipeline (`/api/bridge/ingest`)
2. A virtual mesh simulator (no Bluetooth hardware required)
3. A professional real-time dashboard at `http://localhost:8080/`

---

## Table of Contents

1. [Architecture at a glance](#architecture-at-a-glance)
2. [How to run](#how-to-run)
3. [Dashboard demo flow](#dashboard-demo-flow)
4. [API reference](#api-reference)
5. [Security and correctness model](#security-and-correctness-model)
6. [Testing](#testing)
7. [Project structure](#project-structure)
8. [Productionization roadmap](#productionization-roadmap)
9. [Troubleshooting](#troubleshooting)

---

## Architecture at a glance

```text
Offline sender -> encrypted MeshPacket -> gossip across devices -> bridge uploads -> backend ingest pipeline

Pipeline:
1) SHA-256(ciphertext)
2) idempotency claim (atomic putIfAbsent)
3) decrypt (RSA-OAEP unwrap + AES-GCM verify)
4) freshness validation
5) transactional settlement (debit/credit + ledger)
```

Core guarantees:

- **No intermediate can read plaintext**
- **Tampering fails authentication**
- **Same packet settles once, duplicates are dropped**

---

## How to run

### Prerequisites

- Java **25 LTS**
- Git
- Browser (Chrome/Edge/Firefox/Safari)

Maven is bundled via wrapper (`mvnw`, `mvnw.cmd`).

### Start application (Windows)

```cmd
.\mvnw.cmd spring-boot:run
```

### Start application (macOS/Linux)

```bash
./mvnw spring-boot:run
```

Open:

- Dashboard: `http://localhost:8080/`
- H2 Console: `http://localhost:8080/h2-console`

> Template caching is disabled (`spring.thymeleaf.cache=false`) for faster UI iteration.

### Stop

`Ctrl + C` in terminal.

---

## Dashboard demo flow

1. **Compose & Inject Payment** (`/api/demo/send`)
2. **Run Gossip Round** (`/api/mesh/gossip`) 2–3 times
3. **Bridges Upload to Backend** (`/api/mesh/flush`)
4. Observe:
   - device propagation in mesh panel
   - ledger updates in transaction table
   - idempotency behavior in repeated flushes

---

## API reference

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Dashboard UI |
| GET | `/api/server-key` | Server RSA public key |
| GET | `/api/accounts` | Account balances |
| GET | `/api/transactions` | Recent transactions |
| GET | `/api/mesh/state` | Virtual mesh state |
| POST | `/api/demo/send` | Build + encrypt + inject packet |
| POST | `/api/mesh/gossip` | One gossip round |
| POST | `/api/mesh/flush` | Bridge uploads |
| POST | `/api/mesh/reset` | Reset mesh + idempotency cache |
| POST | `/api/bridge/ingest` | Production-shape bridge ingest endpoint |

### `/api/bridge/ingest` request (example)

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-ciphertext"
}
```

### Response (example)

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible outcomes: `SETTLED`, `DUPLICATE_DROPPED`, `INVALID`.

---

## Security and correctness model

### 1) Untrusted intermediaries

- Payload encrypted with server public key (hybrid model)
- AES-GCM provides integrity (auth tag)

### 2) Duplicate storm handling

- Hash is claimed **before decryption/settlement**
- Atomic claim ensures only first delivery proceeds

### 3) Replay protection

- Encrypted timestamp freshness window
- Nonce per instruction
- Ciphertext hash dedupe

---

## Testing

Run all tests:

```cmd
.\mvnw.cmd test
```

Highlight test:

```cmd
.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

This validates concurrent multi-bridge duplicate delivery settles exactly once.

---

## Project structure

```text
src/main/java/com/demo/upimesh/
├── controller/      # REST + dashboard controller
├── service/         # mesh simulation, ingestion, idempotency, settlement
├── crypto/          # RSA/AES hybrid cryptography
├── model/           # JPA entities and repositories
└── config/          # app configuration

src/main/resources/
├── templates/dashboard.html
└── application.properties
```

---

## Productionization roadmap

Replace demo infrastructure with production equivalents:

- H2 -> PostgreSQL/MySQL
- ConcurrentHashMap idempotency -> Redis (`SET NX EX`)
- Ephemeral RSA keys -> KMS/HSM + rotation
- Simulated mesh -> real BLE/Wi-Fi Direct client implementation
- Open ingest endpoint -> mTLS / signed bridge identities
- Local logging -> centralized observability stack

---

## Troubleshooting

- **PowerShell cannot find mvnw.cmd**
  - Use: `.\mvnw.cmd spring-boot:run`

- **Port 8080 already in use**
  - Change `server.port` in `src/main/resources/application.properties`

- **Dashboard not updating**
  - Hard refresh (`Ctrl+F5`) and ensure app restarted

---

## Disclaimer

This is an educational and portfolio-grade simulation of offline routed deferred settlement. It is not a production banking system.
