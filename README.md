# 📡 UPI Offline Mesh — Demo

A Spring Boot backend that demonstrates offline UPI payments routed through a Bluetooth-style mesh network.

You're in a basement with zero connectivity. You send your friend ₹500. Your phone encrypts the payment, broadcasts it to nearby phones, and the packet hops device-to-device until some phone walks outside, gets 4G, and silently uploads it to this backend. The backend decrypts, deduplicates, and settles.

This repo is the server side of that system, plus a software simulator of the mesh so you can demo the whole flow on a single laptop without any real Bluetooth hardware.

---

## Table of Contents

1. [What this demo proves](#what-this-demo-proves)
2. [How to run it](#how-to-run-it)
3. [The demo flow (step by step)](#the-demo-flow-step-by-step)
4. [Architecture](#architecture)
5. [The three hard problems and how they're solved](#the-three-hard-problems-and-how-theyre-solved)
6. [File-by-file walkthrough](#file-by-file-walkthrough)
7. [API reference](#api-reference)
8. [Tests](#tests)
9. [What's NOT real (and what would change for production)](#whats-not-real-and-what-would-change-for-production)
10. [Honest limitations of the concept](#honest-limitations-of-the-concept)
11. [Troubleshooting](#troubleshooting)

---

## What this demo proves

The system shows three things working end to end:

1. **A payment can travel from sender to backend through untrusted intermediaries** without any of them being able to read or tamper with it. (Hybrid RSA + AES-GCM encryption.)
2. **Even if the same payment reaches the backend simultaneously through multiple bridge nodes, it settles exactly once.** (Idempotency via atomic compare-and-set on the ciphertext hash.)
3. **A tampered or replayed packet is rejected** before it touches the ledger.

You'll see all three in the dashboard.

---

## How to run it

### Prerequisites

- **JDK 17 or newer** installed and on PATH (or `JAVA_HOME` set). Check with `java -version`.
- That's it. No database, no Redis, no Maven install needed (wrapper handles Maven).

### Run on Windows

Open a terminal in the project folder and run:

```cmd
.\mvnw.cmd spring-boot:run
```

The first run downloads Maven (~10 MB) and all dependencies (~80 MB) — give it a couple of minutes. Subsequent runs start in a few seconds.

### Run on Mac/Linux

```bash
./mvnw spring-boot:run
```

### Open the dashboard

Once you see `Started UpiMeshApplication in X.XXX seconds`, open:

**http://localhost:8080**

You'll get a dark dashboard with everything you need to drive the demo.

### Stop the server

`Ctrl + C` in the terminal.

### Run the tests

```cmd
.\mvnw.cmd test
```

The interesting one is `IdempotencyConcurrencyTest` — it fires three threads delivering the same packet simultaneously and asserts that exactly one settles.

---

## The demo flow (step by step)

The dashboard has four buttons that walk through the full pipeline. The intended sequence:

### Step 1 — Compose a payment

Choose sender, receiver, amount, PIN. Click **"📤 Inject into Mesh"**.

**What actually happens on the backend:**

- The server pretends to be the sender's phone.
- It builds a `PaymentInstruction` with a unique nonce and current timestamp.
- It encrypts that with the server's RSA public key (using hybrid encryption — see below).
- It wraps the ciphertext in a `MeshPacket` with a TTL of 5.
- It hands the packet to `phone-alice`, an offline virtual device.

You'll see `phone-alice` now holds 1 packet.

### Step 2 — Run gossip rounds

Click **"🔄 Run Gossip Round"**. Then click it again.

Each round, every device that holds a packet broadcasts it to every other device within "Bluetooth range" (which, in our simulator, means everyone). TTL decrements per hop.

After 1 round: every device holds the packet. After 2 rounds: still every device — TTL is just lower.

In the real system this would happen organically as people walk past each other in the basement.

### Step 3 — Bridge node walks outside

Click **"📡 Bridges Upload to Backend"**.

`phone-bridge` is the only device with `hasInternet=true`. The dashboard simulates that phone walking outside and getting 4G. It POSTs every packet it holds to `/api/bridge/ingest`.

The backend pipeline runs:

1. Hash the ciphertext (`SHA-256`).
2. Try to claim the hash in the idempotency cache.
3. If claimed: decrypt with the server's RSA private key.
4. Verify freshness (`signedAt` within 24 hours).
5. Run the debit/credit in a single DB transaction.

Watch the **Account Balances** table — money has moved. Watch the **Transaction Ledger** — a new row appears.

### Step 4 — Demonstrate idempotency (the killer feature)

Reset the mesh. Inject a single packet. Run gossip 2 times. Now all 5 devices hold the same packet, including multiple bridges in a more complex setup.

To really see idempotency in action, modify `MeshSimulatorService.java` to seed multiple bridge devices, or just:

1. Click "Inject" once.
2. Click "Gossip" twice.
3. Click "Flush Bridges" — only `phone-bridge` is a bridge in the default seed, so just one upload happens.

To exercise the concurrent duplicate case properly, run:

```cmd
.\mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

This test creates one packet, fires 3 threads at `BridgeIngestionService.ingest()` simultaneously, and verifies that exactly one settles, two are dropped as duplicates, and the sender is debited exactly once.

---

## Architecture

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                         SENDER PHONE (offline)                          │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ encrypt with server's RSA public key                     │
│   MeshPacket { packetId, ttl, createdAt, ciphertext }                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ Bluetooth gossip
                                       ▼
        ┌─────────┐  hop   ┌─────────┐  hop   ┌─────────┐
        │stranger1│ ─────▶ │stranger2│ ─────▶ │ bridge  │ ◀── walks outside
        └─────────┘        └─────────┘        └────┬────┘     gets 4G
                                                   │
                                                   ▼ HTTPS POST
┌─────────────────────────────────────────────────────────────────────────┐
│                     SPRING BOOT BACKEND (this project)                  │
│                                                                         │
│  /api/bridge/ingest                                                     │
│       │                                                                 │
│       ▼                                                                 │
│  [1] hash ciphertext (SHA-256)                                          │
│       │                                                                 │
│       ▼                                                                 │
│  [2] IdempotencyService.claim(hash)  ◀── atomic putIfAbsent (≈ Redis    │
│       │                                  SETNX). Duplicates rejected    │
│       │                                  here, before any work.         │
│       ▼                                                                 │
│  [3] HybridCryptoService.decrypt(ciphertext)                            │
│       │       (RSA-OAEP unwraps AES key, AES-GCM decrypts payload       │
│       │        AND verifies the auth tag — tampering = exception)       │
│       ▼                                                                 │
│  [4] Freshness check: signedAt within last 24h                          │
│       │                                                                 │
│       ▼                                                                 │
│  [5] SettlementService.settle()                                         │
│       @Transactional: debit sender, credit receiver, write ledger       │
│       @Version on Account = optimistic locking (defense in depth)       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The three hard problems and how they're solved

### Problem 1: Untrusted intermediates

A random stranger's phone is carrying your transaction. How do you stop them from reading the amount or changing it?

**Solution: Hybrid encryption (RSA-OAEP + AES-GCM).**

The sender encrypts the payload with the server's public key. Only the server holds the private key, so intermediates see opaque ciphertext.

But RSA can only encrypt small data (~245 bytes for a 2048-bit key), and our payload is JSON that could exceed that. So we use the standard hybrid pattern:

1. Generate a fresh AES-256 key for this packet.
2. Encrypt the JSON with AES-256-GCM (fast + authenticated).
3. Encrypt just the AES key with RSA-OAEP.
4. Concatenate: `[256 bytes RSA-encrypted AES key][12 bytes IV][AES ciphertext + 16-byte GCM tag]`.

**Why GCM specifically?** It's authenticated encryption. If an intermediate flips one bit anywhere in the ciphertext, decryption throws an exception — the GCM tag won't verify. The server cannot be tricked into processing tampered data.

This is the same scheme TLS uses. See `HybridCryptoService.java`.

### Problem 2: The duplicate-storm

Three bridge nodes hold the same packet. They all walk outside at the same instant. They all POST to `/api/bridge/ingest` within milliseconds of each other. If you naively process all three, the sender is debited ₹1500 instead of ₹500.

**Solution: Atomic compare-and-set on the ciphertext hash.**

The very first thing the server does on receiving a packet is compute `SHA-256(ciphertext)` and try to claim that hash:

```java
// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;  // true = first claimer, false = duplicate
```

`ConcurrentHashMap.putIfAbsent` is atomic. Even if 100 threads call it at the exact same nanosecond, exactly one returns `null` (the first claimer) and the rest return the existing entry. Only the first claimer proceeds to decrypt and settle. The rest are short-circuited as `DUPLICATE_DROPPED`.

**Why hash the ciphertext, not the packetId or the cleartext?**

- `packetId` can be rewritten by a malicious intermediate. Two copies of the same payment could have different packetIds.
- The cleartext requires decryption first. We want to dedupe before spending CPU on RSA.
- The ciphertext is authenticated by GCM, so any tampering is detectable on decrypt.

In production this `ConcurrentHashMap` becomes Redis:

```text
SET key NX EX 86400
```

There's also a defense-in-depth fallback: `transactions.packet_hash` has a unique index. If the cache layer ever fails and two settlements somehow try to write the same hash, the database rejects the second one.

### Problem 3: Replay attacks

An attacker who captured a ciphertext weeks ago could replay it whenever convenient.

**Solution: Two layers.**

1. Inside the encrypted payload, the sender includes `signedAt` (epoch millis). The server rejects any packet older than 24 hours. The attacker can't change `signedAt` without breaking the GCM tag.
2. Inside the encrypted payload, the sender includes a nonce (UUID). Even if Alice legitimately sends Bob ₹100 twice, the nonces differ → ciphertexts differ → hashes differ → both settle. But a replay of one specific signed packet is byte-identical, so the idempotency cache catches it.

See `BridgeIngestionService.java` for the freshness check.

---

## File-by-file walkthrough

```text
upi-offline-mesh/
├── pom.xml                                  Maven build, Spring Boot, Java 17+
├── mvnw, mvnw.cmd                           Maven wrapper (no install needed)
├── README.md                                this file
└── src/main/
    ├── resources/
    │   ├── application.properties           H2 config, port 8080, TTLs
    │   └── templates/dashboard.html         interactive demo UI
    └── java/com/demo/upimesh/
        ├── UpiMeshApplication.java          Spring Boot main class
        │
        ├── model/                           domain layer
        │   ├── Account.java                 JPA entity, @Version optimistic lock
        │   ├── AccountRepository.java       Spring Data JPA
        │   ├── Transaction.java             settled ledger, unique packet hash index
        │   ├── TransactionRepository.java   Spring Data JPA
        │   ├── MeshPacket.java              wire format wrapper
        │   └── PaymentInstruction.java      decrypted payload object
        │
        ├── crypto/                          cryptography layer
        │   ├── ServerKeyHolder.java         RSA keypair holder
        │   └── HybridCryptoService.java     RSA-OAEP + AES-GCM + hash
        │
        ├── service/                         business logic
        │   ├── DemoService.java             packet creation simulation
        │   ├── VirtualDevice.java           virtual phone model
        │   ├── MeshSimulatorService.java    gossip protocol simulator
        │   ├── IdempotencyService.java      hash claim service
        │   ├── SettlementService.java       transactional ledger settlement
        │   └── BridgeIngestionService.java  hash→claim→decrypt→freshness→settle
        │
        ├── controller/                      HTTP layer
        │   ├── ApiController.java           REST endpoints
        │   └── DashboardController.java     serves dashboard at /
        │
        └── config/
            └── AppConfig.java               scheduling/config helpers

src/test/java/com/demo/upimesh/
└── IdempotencyConcurrencyTest.java          concurrency + tamper tests
```

---

## API reference

| Method | Path | What it does |
|---|---|---|
| GET | `/` | Dashboard HTML |
| GET | `/api/server-key` | Server RSA public key (base64) |
| GET | `/api/accounts` | All accounts and balances |
| GET | `/api/transactions` | Last 20 transactions |
| GET | `/api/mesh/state` | Current state of every virtual device |
| POST | `/api/demo/send` | Simulate sender phone — encrypt + inject packet |
| POST | `/api/mesh/gossip` | Run one round of gossip across the mesh |
| POST | `/api/mesh/flush` | Bridges with internet upload to backend (parallel) |
| POST | `/api/mesh/reset` | Clear mesh + idempotency cache |
| POST | `/api/bridge/ingest` | The production endpoint. Real bridges POST here |
| GET | `/h2-console` | Browse database |

### Request format for `/api/bridge/ingest`

```http
POST /api/bridge/ingest
Content-Type: application/json
X-Bridge-Node-Id: phone-bridge-42
X-Hop-Count: 3

{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "base64-encoded-RSA-and-AES-blob"
}
```

### Response

```json
{
  "outcome": "SETTLED",
  "packetHash": "a3f8c9...",
  "reason": null,
  "transactionId": 42
}
```

Possible `outcome` values: `SETTLED`, `DUPLICATE_DROPPED`, `INVALID`.

---

## Tests

Run all tests:

```cmd
.\mvnw.cmd test
```

Included tests:

- `encryptDecryptRoundTrip` — verifies hybrid encryption round-trip correctness.
- `tamperedCiphertextIsRejected` — flips a ciphertext byte and verifies `INVALID`.
- `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` — verifies exactly-once settlement under parallel duplicate delivery.

---

## What's NOT real (and what would change for production)

This is a teaching demo. To make it production-grade you'd swap these pieces:

| What's in the demo | What it would be in production |
|---|---|
| H2 in-memory/file DB | PostgreSQL / MySQL with replicas |
| `ConcurrentHashMap` for idempotency | Redis with `SET NX EX` |
| RSA keypair regenerated on startup | Private key in HSM/KMS + rotation policy |
| Server-side `DemoService.createPacket()` | Similar logic on actual mobile app client |
| Software-simulated mesh | Real BLE GATT or Wi-Fi Direct transport |
| Open `/api/bridge/ingest` in demo | mTLS / signed bridge certificates |
| In-memory seed accounts | Real KYC'd accounts + secure PIN flow |
| No rate limiting | Velocity checks + per-node rate limits |
| Console logs | Structured logs + SIEM + alerting |

The cryptography and idempotency logic is production-shaped. The main difference is infrastructure hardening.

---

## Honest limitations of the concept

To keep this realistic, here are inherent limitations of fully offline routed payments:

1. **Receiver cannot verify final funds offline.** What they see immediately is an IOU, not guaranteed settlement.
2. **Double-spend risk before settlement.** Sender can issue multiple offline promises; only one may settle.
3. **Real-world BLE behavior is hard.** Background restrictions, connection reliability, and power usage are non-trivial.
4. **Metadata/privacy concerns.** Intermediate devices carrying encrypted packets still reveal traffic metadata.

For projects and demos, it is more accurate to position this as **mesh-routed deferred settlement** rather than instant offline UPI finality.

---

## Troubleshooting

- **`java: command not found`**
  Install JDK 17+ and ensure PATH/JAVA_HOME are set.

- **Port 8080 already in use**
  Change `server.port` in `application.properties`.

- **First `mvnw.cmd` run takes long**
  It is downloading Maven and dependencies on first run.

- **`mvnw.cmd` not recognized in PowerShell**
  Use `\.\mvnw.cmd` prefix, for example:

  ```powershell
  .\mvnw.cmd spring-boot:run
  ```

- **Intermittent concurrency test behavior**
  Re-run test multiple times due to timing sensitivity and machine load.
