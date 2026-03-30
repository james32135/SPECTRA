# SPECTRA Protocol

### *"Your data computes. It never reveals."*

---

> **SPECTRA** is the first **Encrypted Data Intelligence Marketplace** built on Fhenix CoFHE. Organizations contribute encrypted datasets to shared computation pools. Compute buyers run analytics queries — SUM, AVERAGE, MAX, MIN, COUNT_IF — directly on ciphertext. The smart contract computes the result using Fully Homomorphic Encryption, and only the paying buyer can decrypt the output. No party ever sees raw data.

---

## The Problem

**$7.2 trillion** in enterprise data sits locked in silos — not because companies don't want to share it, but because sharing means exposing it.

- **Pharma companies** can't pool patient records for drug discovery without violating HIPAA/GDPR
- **Financial institutions** can't share fraud signals without revealing customer portfolios  
- **AI companies** can't train on private datasets without seeing the data they train on
- **Supply chains** can't verify compliance without exposing pricing and supplier relationships

FHE is the **ONLY** technology that lets you compute on data you cannot read. SPECTRA makes that into a marketplace.

---

## How It Works

SPECTRA creates a three-sided marketplace:

1. **Data Providers** encrypt datasets client-side using `@cofhe/sdk` and deposit encrypted handles on-chain. They earn fees every time their data is used in a computation.

2. **Compute Buyers** submit queries (e.g., "What's the average credit score?"). The smart contract runs FHE operations on the encrypted dataset and returns an encrypted result. The buyer decrypts with their permit.

3. **Validators** attest to data quality without ever seeing raw data — using encrypted range checks.

---

## Documentation

- [Full Architecture](./ARCHITECTURE.md) — System design, data flows, ACL model, threat model
- [5-Wave Strategic Roadmap](./WAVES.md) — Easy-to-hard progression across 5 waves
- [Smart Contract Specifications](./CONTRACTS.md) — Full Solidity code for all contracts
- [FHE Technical Deep-Dive](./FHE_DEEPDIVE.md) — Operations, gas analysis, security properties
- [About](./about.md) — Full project summary

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Encryption** | Fhenix CoFHE (FHE.sol) |
| **Smart Contracts** | Solidity 0.8.25 |
| **Client SDK** | @cofhe/sdk, @cofhe/react |
| **Frontend** | React 18, TypeScript, Vite |
| **Payment Rails** | Privara SDK / ReineiraOS |
| **Chain** | Arbitrum Sepolia |
| **Token Standard** | FHERC20 for $SPEC |

---

## 5-Wave Roadmap Overview

| Wave | App | What It Does |
|------|-----|-------------|
| **1** | SpectraQuery | Protocol design, architecture, and encrypted analytics engine |
| **2** | SpectraInsight | Multi-provider data pools + cross-dataset queries |
| **3** | SpectraML | Privacy-preserving AI inference on encrypted data |
| **4** | SpectraComply | Encrypted KYC/AML + selective disclosure |
| **5** | SpectraOmni | Cross-chain encrypted compute network |

---

## License

MIT
