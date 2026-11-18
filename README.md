# GenomeProof — Analysis Passport Registry

A browser-based application that creates cryptographically signed **Analysis Passports** for genomic studies, fingerprints datasets and result artifacts with SHA-256, and anchors an immutable commitment for each analysis on the Ethereum Sepolia testnet. Anyone can later re-verify an exported passport package independently — with no trust in the original analyst, the hosting server, or this app itself.

## What it does

1. **Create** — an analyst declares the provenance of a genomic analysis run: project and analysis identifiers, dataset fingerprint, reference genome build, pipeline name/version/source, run parameters, environment manifest and the output artifact fingerprint. The browser hashes uploaded files locally (files are never uploaded) and serializes the manifest into a deterministic canonical JSON form.
2. **Register** — the domain-separated commitment `SHA-256("GENOMEPROOF:ANALYSIS_PASSPORT:V1:" ‖ canonical_manifest)` is written to the `AnalysisPassportRegistry` contract on Sepolia through a connected EIP-1193 wallet (MetaMask or compatible). The registration is a real transaction, visible on Etherscan.
3. **Verify** — an exported passport package (JSON) plus the original output artifact are checked: fingerprints are recomputed, the commitment is re-derived, and the on-chain record is read back and compared. Results are reported as `VERIFIED`, `TAMPERED`, `UNREGISTERED`, `UNAUTHORIZED`, `INVALID_PACKAGE` or `INCONCLUSIVE`.
4. **Monitor** — the dashboard derives charts and activity exclusively from actual local records and on-chain `PassportRegistered` events. There is no demo mode and no synthetic data.

## Requirements

- Node.js 20.19+ or 22.12+
- MetaMask or another EIP-1193 wallet for on-chain registration
- Sepolia test ETH for registration transactions
- Any modern browser with WebCrypto support (SHA-256, SubtleCrypto)

## Run locally

```bash
npm install
npm run dev
```

Optional environment configuration (the deployed registry address is the default):

```bash
cp .env.example .env
```

| Variable | Purpose | Default |
| --- | --- | --- |
| `VITE_REGISTRY_ADDRESS` | `AnalysisPassportRegistry` address on Sepolia | `0xc1576223D05dCCB7888446cC7682cE0A5592EC51` |
| `VITE_SEPOLIA_RPC_URL` | Public Sepolia JSON-RPC endpoint for read-only queries | `https://ethereum-sepolia-rpc.publicnode.com` |

## Build, test and lint

```bash
npm run build
npm test
npm run lint
```

## How the passport identity works

- **Canonical serialization** — manifest objects are serialized with recursively sorted keys, no whitespace, NFC-normalized strings, so identical declarations always hash identically across machines.
- **Domain separation** — the commitment is bound to the exact string `GENOMEPROOF:ANALYSIS_PASSPORT:V1:`, preventing cross-protocol hash reuse.
- **On-chain identity** — the passport key is `keccak256(abi.encode(keccak256(projectId), keccak256(analysisId), uint32(version)))`. NFC normalization is applied before hashing, and legacy manifests default to version `1`.
- **Registry contract** — `src/lib/registry-abi.json` holds the exact ABI of the deployed contract. The Solidity source and Hardhat tests under `contracts/` describe a legacy registry and are **not** deployment-compatible; do not redeploy them as a replacement.
- **Immutable records** — registrations cannot be edited or removed; a new version of an analysis is published as a new passport version.

## Registry contract (Sepolia)

```
Address:  0xc1576223D05dCCB7888446cC7682cE0A5592EC51
Network:  Sepolia (chainId 11155111)
Key API:  registerPassport(projectId, passportId, version, commitment)
          getPassport(projectId, passportId, version)
          PassportRegistered event
Access:   registrations restricted to accounts holding REGISTRAR_ROLE;
          the contract administrator can pause registration and manage roles.
```

Admin controls (pause/resume, authorize/revoke registrars) are exposed in **Security & System Info** for wallets holding the admin role.

## Architecture

```
src/
├── routes/            Dashboard, Create, Registered Passports, Verify,
│                      Research Artifacts, Security & System Info
├── components/        App shell, wallet panel, registry admin controls
├── hooks/             Wallet connection, live registry state and events
├── lib/
│   ├── passport.ts    Canonical JSON, SHA-256, commitment, local cache
│   ├── chain.ts       Wallet/RPC access, registration, event polling
│   ├── verify.ts      Independent verification workflow
│   └── registry-abi.json  Exact deployed contract ABI
└── test/              Vitest suites for hashing, encoding, verification, routing
```

Wallet transactions are signed only for what is needed: the registration call itself. Read-only queries use the public RPC endpoint and never require a wallet.

## Data and privacy

- Files are fingerprinted in the browser and never uploaded anywhere.
- The dashboard charts and activity feed use actual records saved in the current browser plus public on-chain events — not generated demo data. The local passport list is a convenience cache; the on-chain registry is authoritative for registration and verification. Data saved in one browser is not automatically shared with another browser.
- Avoid uploading identifiable patient data to public repositories or embedding it in exported manifests; passports should carry fingerprints and metadata only.

## Verifying a passport package

1. In **Create Analysis Passport**, register the passport on-chain (wallet must hold the registrar role and be on Sepolia).
2. Export the verification package from **Registered Passports**.
3. On any machine, open **Verify Passport**, upload the package JSON and the original output artifact.
4. The verifier recomputes both SHA-256 fingerprints and the commitment locally, then compares them against `getPassport(...)` on Sepolia via the public RPC. No wallet is needed to verify.

## Trust boundary

A matching commitment proves only that the declared manifest and supplied artifacts match the registered fingerprints — i.e. that the same inputs and outputs exist as at registration time. It does **not** prove that the analysis was scientifically valid or honestly performed. The registry attests to integrity and provenance, not to scientific correctness.

## License

All rights reserved by the project owner.
