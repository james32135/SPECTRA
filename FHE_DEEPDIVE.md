# SPECTRA Protocol — FHE Technical Deep-Dive

---

## Why FHE, Not Alternatives

### The Fundamental Problem
Traditional encryption (AES, RSA) protects data **at rest** and **in transit**. But to compute on data, you must first decrypt it. FHE is the only technology that eliminates this requirement.

```
TRADITIONAL ENCRYPTION:
Encrypted Data → [DECRYPT] → Plaintext → [COMPUTE] → Result → [ENCRYPT] → Encrypted Result
                     ↑
            EXPOSURE POINT — data is visible during computation

FHE:
Encrypted Data → [COMPUTE ON CIPHERTEXT] → Encrypted Result
                         ↑
              NO EXPOSURE — computation happens on ciphertext
```

### Technology Comparison for Data Marketplaces

| | FHE | ZK-Proofs | MPC | TEE | Federated Learning |
|--|-----|-----------|-----|-----|-------------------|
| **Compute on hidden data** | ✅ Yes | ❌ No (proves facts) | ⚠️ Partial | ⚠️ Trusts hardware | ❌ Model shared |
| **No online coordination** | ✅ Async | ✅ Async | ❌ All online | ✅ Async | ❌ Rounds |
| **No hardware trust** | ✅ Math only | ✅ Math only | ✅ Math only | ❌ Intel/AMD | ✅ Math only |
| **Composable on-chain** | ✅ FHE.sol | ✅ Verifiers | ❌ Off-chain | ❌ Off-chain | ❌ Off-chain |
| **Multi-query on same data** | ✅ Persistent | ❌ Per-proof | ❌ Per-session | ⚠️ Enclave | ❌ Per-round |
| **Data stays encrypted** | ✅ Always | N/A | ❌ Shares | ❌ Plaintext in TEE | ❌ Model shared |

### The SPECTRA Test
> "Remove FHE from SPECTRA. Does it still work?"
> ❌ **No.** Without FHE, the contract cannot compute SUM, AVERAGE, MAX, or COUNT_IF on encrypted records. The raw data would need to be decrypted first, destroying the privacy guarantee that is the product's entire value proposition.

---

## Fhenix CoFHE Architecture in SPECTRA

### Client-Side Encryption Flow

```typescript
import { createCofheClient } from "@cofhe/sdk";

// 1. Initialize CoFHE client
const cofhe = await createCofheClient({
  walletClient,   // from wagmi
  publicClient,   // from wagmi
});

// 2. Provider encrypts salary data before upload
const encryptedSalary = await cofhe.encryptInputs([
  Encryptable.uint64(87500n),  // Salary: $87,500
]);
// Returns: { data: InEuint64[], securityZone: 0 }

// 3. Call contract with encrypted input
await writeContract({
  address: SPECTRA_VAULT_ADDRESS,
  abi: SpectraVaultABI,
  functionName: "depositRecord",
  args: [
    datasetId,
    "salary",
    encryptedSalary.data[0], // InEuint64
  ],
});
```

### On-Chain FHE Computation Flow

```solidity
// Inside SpectraQuery.sol — querySUM function

// 1. Start with encrypted zero
euint64 sum = FHE.asEuint64(0);
FHE.allowThis(sum);  // Contract needs access to compute

// 2. Iterate over encrypted records
for (uint256 i = 0; i < recordCount; i++) {
    euint64 record = vault.getRecord(datasetId, field, i);
    
    // FHE.add() — addition on ciphertext
    // Neither input nor output is ever plaintext
    sum = FHE.add(sum, record);
    FHE.allowThis(sum);  // Maintain access for next iteration
}

// 3. Grant buyer access to result
FHE.allow(sum, msg.sender);
// Only msg.sender (buyer) can now decrypt this handle
```

### Client-Side Decryption Flow

```typescript
// Buyer decrypts query result

// 1. Get or create self-permit (EIP-712 signature)
const selfPermit = await cofhe.permits.getOrCreateSelfPermit();

// 2. Read the encrypted result handle from contract
const resultCtHash = await readContract({
  address: SPECTRA_QUERY_ADDRESS,
  abi: SpectraQueryABI,
  functionName: "queries",
  args: [queryId],
});

// 3. Decrypt via CoFHE threshold network
const plaintext = await cofhe.decryptForView(
  resultCtHash,         // Ciphertext handle
  FheTypes.Uint64,      // Expected type
).execute();

// 4. plaintext = 87500n (the average salary)
// The buyer sees the aggregate result
// But NEVER sees any individual salary
```

---

## FHE Operations Deep-Dive

### 1. `FHE.add(euint64 a, euint64 b) → euint64`
**Used in**: SUM, AVERAGE, COUNT_IF queries
**What it does**: Adds two encrypted values without decrypting either
**SPECTRA usage**: Accumulates encrypted records to compute totals

```solidity
euint64 total = FHE.add(recordA, recordB);
// Neither recordA, recordB, nor total is ever plaintext
```

### 2. `FHE.div(euint64 a, euint64 b) → euint64`
**Used in**: AVERAGE query
**What it does**: Divides encrypted numerator by encrypted denominator
**SPECTRA usage**: `average = sum / count` — both encrypted

### 3. `FHE.max(euint64 a, euint64 b) → euint64`
**Used in**: MAX query  
**What it does**: Returns the larger of two encrypted values
**SPECTRA usage**: Iteratively finds maximum across all records

### 4. `FHE.min(euint64 a, euint64 b) → euint64`
**Used in**: MIN query
**What it does**: Returns the smaller of two encrypted values

### 5. `FHE.gte(euint64 a, euint64 b) → ebool`
**Used in**: COUNT_IF, RANGE queries
**What it does**: Returns encrypted boolean: is a ≥ b?
**SPECTRA usage**: Checks if an encrypted record meets an encrypted threshold

### 6. `FHE.select(ebool cond, euint64 ifTrue, euint64 ifFalse) → euint64`
**Used in**: COUNT_IF, RANGE queries
**What it does**: Encrypted ternary — returns ifTrue when condition is true, ifFalse otherwise
**SPECTRA usage**: `increment = FHE.select(meetsThreshold, one, zero)` — counts matching records without revealing which records match

### 7. `FHE.allowThis(euint64 handle)`
**Used in**: Every contract
**What it does**: Grants the contract itself permission to use this handle in future computations
**SPECTRA usage**: Essential for iterative FHE operations (additions in a loop)

### 8. `FHE.allow(euint64 handle, address to)`
**Used in**: Result delivery
**What it does**: Grants a specific address permission to decrypt this handle
**SPECTRA usage**: Grants buyer decrypt access to query result; grants provider access to their own data

---

## Gas Estimation

| Operation | Estimated Gas | SPECTRA Context |
|-----------|--------------|-----------------|
| `FHE.asEuint64(InEuint64)` | ~50K | Per record upload |
| `FHE.add(a, b)` | ~80K | Per record per SUM query |
| `FHE.div(a, b)` | ~120K | Once per AVERAGE query |
| `FHE.max(a, b)` | ~80K | Per record per MAX query |
| `FHE.gte(a, b)` | ~80K | Per record per COUNT_IF |
| `FHE.select(c, a, b)` | ~100K | Per record per COUNT_IF |
| `FHE.allow(h, addr)` | ~30K | Per result delivery |

### Example: SUM query on 10 records
- 10× `FHE.add` = ~800K gas
- 1× `FHE.allow` = ~30K gas
- Overhead = ~100K gas
- **Total ≈ 930K gas** — well within block limits

### Example: COUNT_IF on 10 records (threshold check)
- 10× `FHE.gte` = ~800K gas
- 10× `FHE.select` = ~1,000K gas
- 10× `FHE.add` = ~800K gas
- **Total ≈ 2.7M gas** — feasible on Arbitrum Sepolia

---

## Async Decryption Pattern

Fhenix CoFHE decryption is **asynchronous** — it requires the threshold network (5 MPC nodes) to collaborate:

```
Buyer calls decryptForView(resultHash)
    │
    ▼
CoFHE SDK sends request to Threshold Network
    │
    ▼
5 MPC nodes each produce a partial decryption
    │
    ▼
Combine partial decryptions → plaintext result
    │
    ▼
Return to buyer's browser (~10-30 seconds on testnet)
```

### UX Handling
SPECTRA handles this with an `AsyncStepper` component:

```
Step 1: "Submitting query..." (TX confirmation)
Step 2: "Computing on encrypted data..." (FHE ops)
Step 3: "Requesting decryption..." (Threshold network)
Step 4: "Result: $87,500" (Plaintext revealed with animation)
```

Each step shows an estimated time and a visual progress indicator to prevent "spinner fatigue."

---

## Security Properties

### 1. Input Privacy
- Data providers encrypt records client-side using `cofhe.encryptInputs()`
- Plaintext **never exists on-chain** — not in calldata, not in state, not in events
- Even failed transactions don't leak data (encrypted inputs are opaque)

### 2. Computation Privacy
- All FHE operations execute inside the CoFHE coprocessor
- The coprocessor sees only ciphertext handles, never raw values
- Intermediate computation results are also encrypted

### 3. Output Privacy
- Query results are encrypted ciphertext handles
- Only the buyer (via `FHE.allow`) can decrypt
- Even the contract owner/admin cannot see results

### 4. Access Control
- `FHE.allowThis()` — contract can compute on data
- `FHE.allow(handle, address)` — specific user can decrypt
- **No `FHE.allowPublic()`** on any raw data or query result
- Only aggregate statistics (post-epoch) may be made public

### 5. Side-Channel Resistance
- `FHE.select()` used instead of `if/else` — gas consumption is identical regardless of which branch is taken
- No `require()` on encrypted values — prevents information leakage through transaction success/failure
