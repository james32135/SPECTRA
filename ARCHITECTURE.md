# SPECTRA Protocol — System Architecture

---

## Overview

SPECTRA is a three-layer protocol that enables encrypted computation-as-a-service on private datasets using Fhenix CoFHE. The architecture separates concerns into: **Data Layer** (encrypted storage + ACL), **Compute Layer** (FHE query execution), and **Market Layer** (pricing, payments, reputation).

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                │
│                                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐ │
│  │ Data Provider │   │ Compute Buyer│   │      Validator           │ │
│  │    dApp       │   │    dApp      │   │       dApp               │ │
│  │              │   │              │   │                          │ │
│  │ @cofhe/sdk    │   │ @cofhe/sdk   │   │   @cofhe/sdk             │ │
│  │ encrypt()     │   │ decrypt()    │   │   attestEncrypted()      │ │
│  └──────┬───────┘   └──────┬───────┘   └────────────┬─────────────┘ │
│         │                  │                         │               │
└─────────┼──────────────────┼─────────────────────────┼───────────────┘
          │                  │                         │
          ▼                  ▼                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      SMART CONTRACT LAYER                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    SpectraRegistry.sol                         │  │
│  │  - registerDataset(encryptedSchema, pricePerQuery)            │  │
│  │  - updateAccess(datasetId, address, permission)               │  │
│  │  - getDatasetMetadata(datasetId) → public schema, prices      │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    SpectraVault.sol                            │  │
│  │  - depositData(datasetId, InEuint64[] encRecords)             │  │
│  │  - Stores all records as euint64 ciphertext handles           │  │
│  │  - ACL: FHE.allowThis() for contract compute access           │  │
│  │  - ACL: FHE.allow(handle, provider) for provider decrypt      │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   SpectraCompute.sol                           │  │
│  │  - submitQuery(datasetId, queryType, params)                  │  │
│  │  - Executes FHE operations on encrypted dataset:              │  │
│  │    • AGGREGATE: FHE.add() across all records                  │  │
│  │    • AVERAGE:   FHE.add() then FHE.div(sum, count)            │  │
│  │    • FILTER:    FHE.gte() + FHE.select() for range queries    │  │
│  │    • COUNT_IF:  FHE.select(condition, 1, 0) + FHE.add()       │  │
│  │    • MAX/MIN:   FHE.max() / FHE.min() across records          │  │
│  │  - Result stored as euint64, ACL: FHE.allow(result, buyer)    │  │
│  │  - Buyer decrypts via decryptForView()                        │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  SpectraMarket.sol                             │  │
│  │  - payForQuery(datasetId, queryId) → escrow $SPEC             │  │
│  │  - claimEarnings(datasetId) → provider withdraws fees         │  │
│  │  - stakeForValidation(amount) → validators stake $SPEC        │  │
│  │  - disputeResult(queryId) → challenge bad data                │  │
│  │  - Privara SDK integration for stablecoin settlement          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  SpectraACL.sol (shared)                       │  │
│  │  - Role management: PROVIDER, BUYER, VALIDATOR, AUDITOR       │  │
│  │  - _grantDecrypt(handle, address) → FHE.allow()               │  │
│  │  - _retainAccess(handle) → FHE.allowThis()                    │  │
│  │  - _makePublic(handle) → FHE.allowPublic()                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                 SpectraToken.sol ($SPEC FHERC20)               │  │
│  │  - Encrypted balances for all holders                         │  │
│  │  - Transfer, approve, allowance — all on ciphertext           │  │
│  │  - Minting for data provider rewards                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
          │                  │                         │
          ▼                  ▼                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FHENIX CoFHE COPROCESSOR                          │
│                                                                      │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────┐  │
│  │ Task     │  │  ZK          │  │   Threshold    │  │  CT      │  │
│  │ Manager  │  │  Verifier    │  │   Network      │  │ Registry │  │
│  │          │  │              │  │   (MPC Decrypt) │  │          │  │
│  └──────────┘  └──────────────┘  └────────────────┘  └──────────┘  │
│                                                                      │
│  All FHE operations execute here — contracts submit tasks,           │
│  coprocessor returns encrypted results asynchronously                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow — Complete Query Lifecycle

```
 DATA PROVIDER                    SMART CONTRACT                    COMPUTE BUYER
 ─────────────                    ──────────────                    ─────────────
      │                                │                                │
      │  1. Encrypt dataset            │                                │
      │     client-side                │                                │
      │  cofhe.encryptInputs([         │                                │
      │    Encryptable.uint64(salary), │                                │
      │    Encryptable.uint64(age),    │                                │
      │    Encryptable.uint64(score)   │                                │
      │  ])                            │                                │
      │                                │                                │
      │  2. depositData(records)  ────▶│                                │
      │                                │  3. Store as euint64 handles   │
      │                                │     FHE.allowThis(each handle) │
      │                                │     FHE.allow(handle, provider)│
      │                                │                                │
      │                                │                                │
      │                                │  4. submitQuery(              │◀──── Buyer
      │                                │       datasetId,              │
      │                                │       AGGREGATE,              │
      │                                │       {field: "salary"}       │
      │                                │     )                         │
      │                                │                                │
      │                                │  5. FHE COMPUTATION:          │
      │                                │     result = FHE.asEuint64(0) │
      │                                │     for each record:          │
      │                                │       result = FHE.add(       │
      │                                │         result,               │
      │                                │         records[i].salary     │
      │                                │       )                       │
      │                                │     FHE.allowThis(result)     │
      │                                │     FHE.allow(result, buyer)  │
      │                                │                                │
      │                                │  6. Query result stored ──────▶│
      │                                │     as euint64 handle          │
      │                                │                                │
      │                                │                                │  7. Buyer decrypts:
      │                                │                                │     cofhe.permits
      │                                │                                │       .getOrCreateSelfPermit()
      │                                │                                │     cofhe.decryptForView(
      │                                │                                │       resultHash,
      │                                │                                │       FheTypes.Uint64
      │                                │                                │     ).execute()
      │                                │                                │
      │  8. Provider earns fees ◀──────│  payForQuery() escrow ◀───────│
      │     claimEarnings()            │                                │
      │                                │                                │
```

---

## Access Control Model

SPECTRA uses a layered ACL model that maps directly to Fhenix CoFHE primitives:

| Data | Who can decrypt | FHE primitive |
|------|----------------|---------------|
| Raw dataset records | Provider only (own data) | `FHE.allow(handle, provider)` |
| | Contract (for computation) | `FHE.allowThis(handle)` |
| Query result | Buyer only | `FHE.allow(result, buyer)` |
| Validation attestation | Validator + Provider | `FHE.allow(attestation, validator)` |
| Pool statistics (post-epoch) | Public (aggregates only) | `FHE.allowPublic(aggregate)` |
| Individual records | NEVER public | No `allowPublic` on raw data |

### Key Security Invariants

1. **Raw data never becomes public** — `FHE.allowPublic()` is NEVER called on individual records
2. **Query results are buyer-exclusive** — only the paying buyer can decrypt the result
3. **Validators cannot read data** — they run encrypted range checks (`FHE.gte()`, `FHE.lte()`) that return `ebool`, not raw values
4. **Cross-query privacy** — two buyers querying the same dataset get independent ciphertext handles; knowing your result reveals nothing about anyone else's query
5. **Contract always retains access** — `FHE.allowThis()` on every stored ciphertext handle, so future computations can operate on older data

---

## Query Types & FHE Operations

| Query Type | FHE Operations Used | Example |
|-----------|--------------------:|---------|
| **SUM / AGGREGATE** | `FHE.add()` | "Total salary expenditure" |
| **AVERAGE** | `FHE.add()` + `FHE.div()` | "Average credit score" |
| **COUNT_IF** | `FHE.gte()` + `FHE.select()` + `FHE.add()` | "How many users have score > 700?" |
| **MAX / MIN** | `FHE.max()` / `FHE.min()` | "Highest transaction value" |
| **RANGE FILTER** | `FHE.gte()` + `FHE.lte()` + `FHE.select()` | "Sum of salaries between 50K–100K" |
| **WEIGHTED SCORE** | `FHE.mul()` + `FHE.add()` + `FHE.div()` | "Weighted risk score" |
| **THRESHOLD CHECK** | `FHE.gte()` → `ebool` | "Does this dataset meet minimum quality?" |

---

## Integration Points

### Privara / ReineiraOS Integration
- **Payment escrow**: Query fees held in `ConfidentialEscrow` from Privara SDK
- **Stablecoin settlement**: Providers paid in USDC via ReineiraOS settlement rails
- **Programmable release**: Fees released to provider only after query result is delivered and undisputed

### FHERC20 Integration
- **$SPEC token**: Protocol governance + staking + payment
- **Encrypted balances**: All $SPEC holdings are encrypted — nobody knows your balance
- **Staking**: Validators stake $SPEC to earn validation fees; slashed on bad attestation

---

## Threat Model

| Threat | Mitigation |
|--------|-----------|
| **Data buyer sees raw data** | IMPOSSIBLE — they only receive FHE computation results, never individual records |
| **Contract reveals data in logs** | No plaintext ever exists in contract — only euint64 handles |
| **Validator colludes with buyer** | Validators see only ebool attestation results, not raw data |
| **Provider submits garbage data** | Validators run encrypted quality checks; bad data → slash stake |
| **Replay attack on queries** | Per-query nonce + unique result handles prevent reuse |
| **Block explorer exposes state** | All storage is ciphertext — explorer shows only handle hashes |
| **Front-running queries** | Query parameters encrypted client-side before submission |
