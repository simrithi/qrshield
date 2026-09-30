# QRSHIELD: End-to-End Engineering Architecture Report

### Tamper-Proof UPI QR Verification & Multi-Bank Fraud Flags on NPCI DRUNIX

> Built for the **DRUNIX Hackathon 2026 in collaboration with Citi** (India Blockchain Forum, CHL-7007)
> Track: **Real-Time Payments** · Themes: **Fraud Detection** · **Financial Inclusion**
> Team: **TrustLayer**

---

## Project Contributors

| Member # | Student Name | Registration Number | Department |
|---|---|---|---|
| Member 1 | Simrithi S | RA2411003020843 | B.Tech (3rd Year) • Computer Science & Engineering, SRM IST Ramapuram |

---

## Executive Summary

**QRShield** is a permissioned-blockchain trust layer for UPI QR payments. Static UPI QR codes are plain, unsigned text, so a fraudster can paste their own QR over a merchant's code and the payer's app cannot tell the difference. QRShield makes every merchant QR **cryptographically signed, registered on a shared DRUNIX ledger, verifiable at scan time and instantly revocable** across every connected app. When a swapped QR is reported, the fraudster's UPI ID is shared across banks as a salted hash and confirmed only when **2 of 3 banks agree**.

QRShield delivers three capabilities in one focused product:

| # | Capability | What it does |
|---|---|---|
| 01 | **Prove** | Every merchant QR is signed by the acquiring bank and registered on DRUNIX with NPCI endorsement |
| 02 | **Revoke** | A reported or swapped QR is revoked in one transaction and blocked in every app |
| 03 | **Block** | The fraudster's UPI ID is flagged across banks as a hash, confirmed by 2-of-3 endorsement |

The architecture decouples responsibilities across four tiers:

- **Client Presentation Layer:** Android payer and merchant apps (Kotlin, CameraX, ML Kit, NFC), an Angular bank operations dashboard and an LLM voice assistant.
- **Organization Service Layer:** Java Spring Boot services per organization (Bank A, Bank B, NPCI) using the Fabric Gateway SDK, with ECDSA signing and a Kafka payment stream.
- **AI Scoring Layer:** Python service (XGBoost, Isolation Forest, NetworkX, SHAP) for QR-swap and fraud-pattern detection.
- **Distributed Ledger Layer:** NPCI DRUNIX (Hyperledger Fabric based) three-organization network running Java chaincode, with private data collections and a YugabyteDB SQL state database.

---

## 1. Threat Model

| # | Attack | Mechanism | QRShield Defense |
|---|---|---|---|
| 1 | Sticker Swap | Fraudster's QR pasted over a merchant's genuine QR | Signature + registry check fails → **Do not pay** |
| 2 | Fake Public QRs | QRs on parking meters, EV chargers, posters, fake fine notices | Unregistered or out-of-geofence QR → **Check first / Do not pay** |
| 3 | Scan-to-Receive Scam | QR actually requests a payment; victim enters PIN | Personal account and payment-request status shown before paying |
| 4 | Screen Tampering | Malware alters dynamic QR on billing screen or POS | Payload signature mismatch detected at scan time |
| 5 | QRLjacking | Login QR relayed to hijack a session | Unregistered origin flagged; signed payloads only |

---

## 2. Client Presentation Layer

### 2.1 Technical Stack
- **Payer App:** Android (Kotlin), CameraX for capture, ML Kit barcode scanning, NFC reader.
- **Merchant App:** Android daily self-check of the merchant's own QR against the registry.
- **Bank Ops Dashboard:** Angular SPA for QR registrations, reports, flags and audit trail.
- **Voice Assistant:** LLM with MCP tool calls over DRUNIX, Tamil and Hindi speech.

### 2.2 Payer Verification States

| State | Condition | Displayed Result |
|---|---|---|
| **VERIFIED** | Valid signature, registered, active, within geofence | "Verified: Ravi Tea Stall, 20 m away" |
| **CHECK FIRST** | Valid UPI ID but not a registered merchant | "Personal account, not a merchant" |
| **DO NOT PAY** | Revoked, reported, signature invalid, or far outside geofence | "QR revoked" / "Registered 40 km away" |

Every state pairs an icon with plain words (never color alone) and is available in Tamil, Hindi and English.

### 2.3 Physical Tamper Layer
- **VOID tamper-evident sticker** with a printed serial that must match the ledger record.
- **NFC tag in the QR stand** carrying the same signed payload for tap-to-cross-check.
- **Rotating e-ink QR** (optional, high-value counters) with a signed payload that refreshes so photographed copies expire.

---

## 3. Organization Service Layer

### 3.1 Technical Stack
- **Runtime:** Java 17 + Spring Boot, one service per organization.
- **Ledger Client:** Hyperledger Fabric Gateway SDK (Java).
- **Cryptography:** ECDSA signing of QR payloads by the acquiring bank; SHA-256 salted hashing of UPI IDs.
- **Streaming:** Apache Kafka topic simulating a real-time UPI payment stream.
- **AI Scoring:** Python service (XGBoost, Isolation Forest, NetworkX graph features, SHAP explanations) called before payment release.

### 3.2 REST API Architecture

| Method | Endpoint | Caller | Description |
|---|---|---|---|
| POST | `/api/qr/register` | Acquiring Bank | Signs payload and submits `RegisterQR` |
| GET | `/api/qr/verify/{serial}` | Payer App | Signature check + `VerifyQR` query |
| POST | `/api/qr/{serial}/revoke` | Acquiring Bank / NPCI | Submits `RevokeQR` |
| POST | `/api/qr/{serial}/report` | Merchant / Payer | Submits `ReportQR` |
| POST | `/api/flags` | Member Bank | Submits `FlagAccount` with hashed UPI ID |
| POST | `/api/flags/{id}/endorse` | Member Bank | Submits `EndorseFlag` |
| GET | `/api/flags/check/{hash}` | Payer Bank | `CheckAccount` before payment release |
| POST | `/api/flags/{id}/appeal` | Account Holder via Bank | Submits `AppealFlag` |
| GET | `/api/health` | Public | Liveness probe |

---

## 4. Distributed Ledger Layer (NPCI DRUNIX)

### 4.1 Network Topology
- **Organizations:** Bank A (acquirer), Bank B (issuer), NPCI (network authority).
- **Peers:** one endorsing peer per organization; Lite Peers serve high-volume read-only verification.
- **State Database:** YugabyteDB (SQL) for geofence and report-pattern queries.

### 4.2 Chaincode Functions (Java)

| Function | Invoked By | Endorsement Policy | Effect |
|---|---|---|---|
| `RegisterQR` | Acquiring Bank | `AND(Acquirer, NPCI)` | Stores QR serial, UPI ID hash, merchant ID, geofence, status = ACTIVE |
| `VerifyQR` | Any App | Query (no ordering) | Returns status, merchant name, geofence |
| `RevokeQR` | Acquirer / NPCI | `AND(Acquirer, NPCI)` | Status = REVOKED; visible to all apps instantly |
| `ReportQR` | Merchant / Payer | Reporter org | Status = UNDER_REVIEW |
| `FlagAccount` | Member Bank | Flagging bank | Adds hashed UPI ID as SUSPECTED |
| `EndorseFlag` | Member Bank | `OutOf(2, BankA, BankB, BankC)` | Status = CONFIRMED on 2-of-3 |
| `CheckAccount` | Payer Bank | Query | Returns flag status for a hash |
| `AppealFlag` | Account Holder via Bank | Holder's bank | Status = UNDER_APPEAL |

### 4.3 Ledger Data Model

| Asset | Key | Fields | Visibility |
|---|---|---|---|
| `QRRecord` | QR serial | upiIdHash, merchantId, merchantName, geofence, status, acquirer, createdAt | Public channel |
| `MerchantKYC` | merchantId | KYC details | Private data collection (Acquirer + NPCI) |
| `FraudFlag` | salted UPI ID hash | status, endorsements[], reason, createdAt | Private data collection (member banks) |
| `Appeal` | appealId | flagId, status, resolution | Member banks |

### 4.4 QR Lifecycle State Machine

```
                 RegisterQR (Acquirer + NPCI)
                             │
                             ▼
                      ┌─────────────┐
          ┌──────────►│   ACTIVE    │───────────────┐
          │  Cleared  └──────┬──────┘   RevokeQR    │
          │                  │ ReportQR             │
          │                  ▼                      ▼
          │          ┌──────────────┐        ┌─────────────┐
          └──────────│ UNDER_REVIEW │───────►│   REVOKED   │
                     └──────────────┘RevokeQR└─────────────┘
```

### 4.5 Fraud Flag State Machine

```
      FlagAccount (1 bank)
               │
               ▼
        ┌─────────────┐   EndorseFlag (2 of 3 banks)   ┌─────────────┐
        │  SUSPECTED  │───────────────────────────────►│  CONFIRMED  │◄──────┐
        └─────────────┘                                └──────┬──────┘       │
                                                              │ AppealFlag   │ Appeal
                                                              ▼              │ rejected
                                                       ┌──────────────┐      │
                                                       │ UNDER_APPEAL │──────┘
                                                       └──────┬───────┘
                                                              │ Appeal upheld
                                                              ▼
                                                       ┌─────────────┐
                                                       │   CLEARED   │
                                                       └─────────────┘
```

### 4.6 Payment Screening Rules (pseudo-logic)

```java
// Rule 1: QR revoked or signature invalid
if (qr.status == REVOKED || !signatureValid) return BLOCK("QR not trusted");

// Rule 2: Receiving UPI ID confirmed by 2 of 3 banks
if (flag.status == CONFIRMED) return BLOCK("Confirmed fraud account");

// Rule 3: Scan location far outside registered geofence
if (distanceFromGeofenceKm > GEOFENCE_LIMIT) return WARN("QR registered elsewhere");

// Rule 4: Not a registered merchant
if (qr == null) return WARN("Personal account, not a merchant");

// Rule 5: Default
return VERIFIED;
```

---

## 5. End-to-End Workflow

```
 Merchant     Acquiring Bank        NPCI        DRUNIX Ledger        Payer App        Payer Bank
    │               │                 │                │                  │                │
    │ Onboard + KYC │                 │                │                  │                │
    │──────────────►│                 │                │                  │                │
    │               │ Sign QR (ECDSA) │                │                  │                │
    │               │ RegisterQR ─────┼───────────────►│                  │                │
    │               │                 │ Endorse ──────►│                  │                │
    │               │                 │                │   Scan + verify  │                │
    │               │                 │                │◄──── VerifyQR ───│                │
    │               │                 │                │ ACTIVE/REVOKED ─►│                │
    │               │                 │                │                  │ Pay ──────────►│
    │               │                 │                │◄──────────── CheckAccount ────────│
    │               │                 │                │                  │◄─ Release /    │
    │               │                 │                │                  │   Warn / Block │
```

### 5.1 Phase Breakdown
1. **Merchant Onboarding:** acquiring bank completes KYC; KYC stored in a private data collection.
2. **QR Signing & Registration:** payload signed and registered with `AND(Acquirer, NPCI)` endorsement.
3. **Scan & Verify:** payer scanner validates the signature and queries the registry via Lite Peers.
4. **Payment Screening:** payer bank checks the fraud-flag registry before release.
5. **Report & Revoke:** swapped QRs are reported and revoked; all apps update instantly.
6. **Multi-Bank Confirmation:** the fraudster's hashed UPI ID is confirmed on 2-of-3 endorsement.
7. **Audit & Appeal:** every registration, flag, appeal and revocation is recorded immutably.

---

## 6. Technology Stack

| Layer | Technology |
|---|---|
| Blockchain | NPCI DRUNIX, three-organization test network |
| Smart Contracts | Java chaincode |
| Backend | Java, Spring Boot, Fabric Gateway SDK, Kafka |
| Security | ECDSA QR signing, salted SHA-256 hashing, private data collections |
| Mobile | Android (Kotlin), CameraX, ML Kit scanning, NFC |
| Dashboard | Angular |
| AI | Python, XGBoost, Isolation Forest, NetworkX, SHAP |
| Data | YugabyteDB state DB, synthetic UPI transaction generator |
| Voice | LLM with MCP tool calls, Tamil and Hindi speech |
| DevOps | Docker, AWS EC2, GitHub Actions |

---

## 7. Build Plan & Verification Matrix

### 7.1 Six-Week Roadmap (Coding Phase: 10 Oct – 22 Nov 2026)

| Week | Dates | Deliverable |
|---|---|---|
| W1 | 10 – 16 Oct | DRUNIX 3-org network + chaincode skeleton |
| W2 | 17 – 23 Oct | ECDSA signing + QR registry |
| W3 | 24 – 30 Oct | Android scan and verify |
| W4 | 31 Oct – 6 Nov | NFC + revocation |
| W5 | 7 – 13 Nov | AI swap detection + fraud flags |
| W6 | 14 – 22 Nov | Demo polish + load test |

### 7.2 Verification Matrix

| Component | Test | Expected Result | Status |
|---|---|---|---|
| Chaincode endorsement | Register QR with only one org signature | Rejected | PLANNED |
| Revocation | Revoke QR, re-scan from a second device | DO NOT PAY shown | PLANNED |
| Signature integrity | Tamper one byte of payload | Verification fails | PLANNED |
| Fraud flag | Flag with 1 of 3 banks | Stays SUSPECTED | PLANNED |
| Fraud flag | Endorse with 2 of 3 banks | CONFIRMED | PLANNED |
| Privacy | Query KYC from non-member org | Access denied | PLANNED |
| Load | Concurrent VerifyQR queries | Latency measured and reported | PLANNED |

*Status will be updated with measured results as each component is built and tested.*

---

## 8. Real vs Simulated

| Real | Simulated |
|---|---|
| DRUNIX network, chaincode, endorsement policies, private data collections | Bank core banking systems |
| QR signing and verification | UPI switch |
| Android apps, Angular dashboard, AI service | Payment data (synthetic generator) |

---

## 9. Future Enhancements
- Voice checks in Tamil and Hindi, including feature phones via UPI 123PAY
- AI mule-account detection shared across banks
- Rotating signed QRs on low-cost e-ink displays
- Offline verification with signed registry snapshots
- Extension to invoice, utility-bill and donation QRs

---

## 10. Evaluation Summary

- **Meaningful DRUNIX Usage:** shared registry, multi-org endorsement, 2-of-3 bank confirmation, private data collections and an immutable audit log are all load-bearing.
- **Privacy by Design:** only salted hashes cross bank boundaries; KYC stays with the acquirer and NPCI.
- **No Single Point of Control:** registration and revocation need two organizations; fraud confirmation needs two of three banks.
- **Inclusive by Default:** icon, color and words on every result, in Tamil, Hindi and English, with no new hardware for merchants.

