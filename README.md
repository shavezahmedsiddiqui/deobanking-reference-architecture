# DeoBanking Reference Architecture

**Bridging Self-Custody, Digital Assets, and Banking Infrastructure**

> **Status:** Proposed reference architecture · **Version:** 0.1.0 · **Updated:** 2026-11-10  
> **Author:** Shavez Ahmed Siddiqui · [Official website](https://shavezahmedsiddiqui.com/)

A technology reference specification for a modular financial interface that connects user-authorized digital asset wallets with accounting infrastructure and **eligible** banking and payment service providers. The design focuses on hybrid custody choices, multi-asset portfolio presentation, policy-controlled payments, and reconciled settlement status.

**DeoBanking** is used here as a *design term* for decentralized-oriented financial application architecture. It is **not** a legal or regulatory classification. This repository contains concepts and sample contracts, **not a deployed bank, licensed financial service, audited smart contract, or production SDK**.

## Contents

- [Design goals](#design-goals)
- [System context and architecture](#system-context-and-architecture)
- [Trust and custody models](#trust-and-custody-models)
- [Component contracts](#component-contracts)
- [Asset and ledger semantics](#asset-and-ledger-semantics)
- [Transaction orchestration](#transaction-orchestration)
- [Wallet-funded merchant payment](#wallet-funded-merchant-payment)
- [Identity, security and governance](#identity-security-and-governance)
- [Failure handling and reconciliation](#failure-handling-and-reconciliation)
- [Verification plan](#verification-plan)
- [Proposed repository structure](#proposed-repository-structure)
- [References](#references)

## Design goals

1. **One coherent interface, multiple asset models.** Present supported fiat accounts, stablecoins and blockchain tokens without treating them as identical financial claims.
2. **Explicit custody semantics.** Represent who can sign transactions, who has recovery authority, and who owes which obligations.
3. **Deterministic authorization.** Execute transfers only after independently evaluated identity, permissions, balance, limits and policy checks.
4. **Provider-neutral integrations.** Isolate the application from institution-specific API protocols and payment routes.
5. **Event-driven financial state.** Track payment progression from authorization to external confirmation and reconciliation.
6. **Auditability.** Maintain traceable ledger postings, correlation identifiers, evidence references and controlled corrections.
7. **Composable services.** Permit modules to evolve independently while preserving enforceable accounting and authorization boundaries.

## System context and architecture
<img width="1550" height="1742" alt="mermaid-diagram (11)" src="https://github.com/user-attachments/assets/77660032-61f4-48b4-bfe2-baaf2cd83450" />
### Responsibility boundaries

| Boundary | Owns | Does not imply |
| --- | --- | --- |
| Customer / smart account | On-chain authorization as defined by its key and recovery policy | A bank deposit or regulated account |
| Wallet adapter | Transaction construction, signing integration and confirmation tracking | Unrestricted rights to sign for the customer |
| Account and position service | Unified view of separately identified asset positions | That all displayed assets are liabilities of one institution |
| Internal ledger | Platform-accounted obligations, fees, reservations and postings | That an external rail has settled |
| Banking / payment partner | Financial services under its actual authorizations and contracts | That an application automatically receives the partner's licenses |
| Orchestration | Workflow coordination and evidentiary state transitions | Cross-rail atomicity |

## Trust and custody models

Custody classification must be derived from **effective control**, not product terminology.

| Model | Signing authority | Recovery / intervention | Accounting implication |
| --- | --- | --- | --- |
| Self-custody | User-controlled key or user-governed smart-account policy | Per the user's selected recovery policy | Blockchain holdings are not automatically platform liabilities |
| Custodial service | Provider or delegated custodian has effective control | Governed by custody agreement and controls | Customer positions and segregation records may be required |
| Hybrid / policy-controlled | Shared or conditional controls; actual authority depends on configuration | Explicit guardians, signers, thresholds and emergency rules | Record obligations separately from purely displayed wallet holdings |

**Architecture rule:** provide `custody_model`, `controller`, `signing_policy_id`, `recovery_policy_id`, `asset_issuer`, and `account_provider` as distinct metadata; never infer custody from the presence of a wallet UI.

Smart-contract accounts can support programmable validation and execution. For an Ethereum implementation, [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) provides a useful standards reference; this document does not mandate it.

## Component contracts

| Service | Required inputs | Outputs / guarantees |
| --- | --- | --- |
| Identity service | Authenticated actor, session context, permissions | Signed/verified principal context; session validity |
| Asset registry | Canonical chain ID, contract/issuer ID, currency | Decimals, asset class, issuer, settlement rules |
| Wallet gateway | Constrained intent, approved network, user signature | Broadcast reference, confirmation evidence, finality policy |
| Banking gateway | Partner-authorized instruction and beneficiary data | Partner reference, status updates, settlement evidence |
| Ledger engine | Journal entry with balanced postings and idempotency key | Atomic, durable postings, journal ID |
| Position service | Bank position, ledger position, wallet position | **Labeled** consolidated view, never fungibility by default |
| Reconciliation | External statements, blockchain events, internal records | Matched entries or exceptions with audit evidence |

### Example: normalized asset position

See [`examples/asset-position.json`](examples/asset-position.json). Amounts are decimal strings for presentation; production integrations should use integer minor/base units with explicit precision metadata.

```json
{
  "position_id": "pos_example_01",
  "asset_id": "eip155:1/erc20:0xEXAMPLE",
  "asset_class": "crypto_asset",
  "custody_model": "self_custody",
  "controller": "customer",
  "source": "blockchain_wallet",
  "quantity": "500.000000",
  "quantity_decimals": 6,
  "valuation_currency": "USD",
  "as_of": "2026-11-10T10:00:00Z",
  "status": "confirmed"
}
```

The contract address above is deliberately **not real** and is not usable for transactions.

### Example: payment intent envelope

```json
{
  "intent_id": "intent_01",
  "idempotency_key": "customer-123-payment-01",
  "principal_id": "customer-123",
  "source": {"type": "self_custody_wallet", "wallet_ref": "wallet_01"},
  "destination": {"type": "merchant_payment", "beneficiary_ref": "merchant_01"},
  "input_asset": "USDC_ON_APPROVED_NETWORK",
  "max_input_base_units": "25000000",
  "payout_currency": "EUR",
  "min_payout_minor_units": "2100",
  "quote_id": "quote_01",
  "quote_expires_at": "2026-11-10T10:01:00Z",
  "authorization_policy_id": "wallet_customer_confirm_v1"
}
```

This is an **illustrative internal schema**, not a request format accepted by any bank, payment network or blockchain.

## Asset and ledger semantics

A unified dashboard must keep asset ownership, issuer claims and platform obligations separate. A displayed portfolio total is a valuation view, not a spendable bank balance.

For entries booked in the platform ledger, each committed journal group must satisfy, **within each currency or unit of account**:

```text
sum(debits) == sum(credits)
```

A transfer of 100 monetary units from Customer A to Customer B *within the same institution's liability ledger* can be represented as:

| Account | Debit | Credit |
| --- | ---: | ---: |
| Customer A liability | 100 | 0 |
| Customer B liability | 0 | 100 |

For fiat-to-token transactions, journal entries should additionally represent any platform liabilities, provider receivables, fees and settlement clearing accounts as applicable to the real contractual arrangement. **Do not** book the customer's independently controlled wallet holdings as platform deposits simply because they are shown in the UI.

Recommended balance concepts:

- `posted`: journal-derived, committed position;
- `reserved`: amount encumbered by active, authorized payment workflows;
- `available`: spendable under account and risk policy;
- `external_confirmed`: a status backed by evidence from the specific settlement rail.

Reservations are not proof of external settlement. Corrections require reversing or adjusting entries rather than silent alteration of the original journal.

## Transaction orchestration

<img width="786" height="1696" alt="mermaid-diagram (12)" src="https://github.com/user-attachments/assets/fb425ea3-31dd-4872-be3c-f5f148ac3b52" />


This is a **canonical orchestration model**. Rail-specific flows may use additional states for reversals, chargebacks, expiry, partial fulfillment and chain reorganization windows.

### Deterministic policy gate (illustrative pseudocode)

```text
function approve(intent, principal, snapshot):
    require authenticated(principal)
    require authorized_for_source(principal, intent.source)
    require approved_destination(intent.destination)
    require quote_not_expired(intent.quote_id)
    require supported_assets_and_networks(intent)
    require within_risk_and_compliance_limits(intent)
    require balance_or_signing_capacity_available(snapshot, intent)
    require valid_independent_approval(intent, principal)
    return APPROVED_INTENT
```

A conversational or AI layer may assist with preparation, but the policy gate and signing authority remain independent.

## Wallet-funded merchant payment

A conceptual stablecoin-funded fiat payout sequence:

1. Customer requests a purchase and receives a bounded quote.
2. Orchestrator validates beneficiary, assets, network, quote expiry and permissions.
3. Customer authorizes a specific token transfer through the configured signing model.
4. Wallet gateway obtains chain transaction evidence under its finality policy.
5. Authorized conversion and payment partners perform their contracted services.
6. Banking gateway receives disbursement confirmation according to the applicable rail.
7. Ledger records relevant obligations and charges; reconciliation binds the independent evidence references.
8. Customer sees **distinct statuses** for digital asset transfer and merchant payout.

<img width="2834" height="1314" alt="mermaid-diagram (13)" src="https://github.com/user-attachments/assets/3f5e2c4c-7378-4caf-aca6-d30083da8039" />

**Important:** Blockchain movement, conversion, and fiat payout are generally **separate financial events**. The reference flow does not promise a cross-system atomic transaction or a particular payout time.

## Identity, security and governance

- Use short-lived credentials and service-to-service authentication; never embed private keys in business-service logs.
- Enforce object- and function-level authorization on every account, wallet and beneficiary endpoint.
- Apply chain, asset, destination, value, nonce and deadline constraints to signed actions.
- Keep custody or recovery permissions explicit in the user's agreement and technical configuration.
- Use allowlists or constrained provider registries for banking and blockchain integrations.
- Preserve tamper-evident audit records, transaction correlation IDs and structured error handling.
- Separate operator access, platform administration and user financial authority.
- Review applicable financial, payments, custody, AML and privacy obligations per jurisdiction and product.

See [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md) for a starter security review checklist.

## Failure handling and reconciliation

| Situation | Expected system behavior |
| --- | --- |
| API retry after an ambiguous network timeout | Resolve using the original idempotency key and persistent transaction state; do not automatically create another payment |
| Blockchain transfer confirmed, payout delayed | Keep payout in a separate pending state with a traceable obligation and partner reference |
| Provider status unknown | Hold the related value-moving retry until authoritative evidence or controlled exception resolution |
| Blockchain reorganization | Apply chain-specific confirmation policy and reconciliation updates; never equate broadcast with finality |
| Bank partner unavailable | Use only eligible configured alternatives, preserving authorization and funding requirements |
| Ledger reconciliation mismatch | Open an exception; correct through auditable postings or documented resolution, not manual balance overwrite |

For all events, retain a correlation chain: `intent_id -> wallet_tx_ref -> provider_ref -> journal_id -> reconciliation_id`.

## Verification plan

The design should be validated with at least these cases before an implementation is described as production-ready:

| Test | Assertion |
| --- | --- |
| Concurrent overspend attempts | At most permitted funds can be reserved or spent |
| Unbalanced journal submission | Posting rejected atomically |
| Duplicate identical payment requests | Exactly one business intent is created |
| Reused key with different request body | Conflict is rejected |
| Unsupported chain / fake token metadata | Signing or settlement rejected |
| Expired quote | Execution blocked |
| Lost provider acknowledgment | No blind duplicate payout |
| Missing blockchain confirmations | No premature finality status |
| Recovery-policy mismatch | Unauthorized signer cannot initiate value movement |
| External statement mismatch | Exception generated and journal preserved |

**Status:** No implementation, performance benchmark, audit, or test result is included in this documentation. The cases above specify intended future verification, not achieved results.

## Proposed repository structure

```text
deobanking-reference-architecture/
├── README.md
├── docs/
│   └── THREAT_MODEL.md
└── examples/
    ├── asset-position.json
    └── payment-intent.json
```

A future implementation could add `schemas/`, `contracts/`, `adapters/`, `tests/` and `benchmarks/` once software and reproducible test results exist. Do not create empty folders merely to imply a more mature codebase.

## References

- [ERC-4337 — Account Abstraction Using Alt Mempool](https://eips.ethereum.org/EIPS/eip-4337)
- [ESMA — MiCA Article 75: Providing Custody and Administration of Crypto-Assets](https://www.esma.europa.eu/publications-and-data/interactive-single-rulebook/mica/article-75-providing-custody-and)
- [OWASP — API Security Top 10](https://owasp.github.io/www-project-api-security/)

## Author and citation

**Shavez Ahmed Siddiqui** — technology research spanning blockchain systems, digital payments, financial infrastructure and wallet architecture. [Official website](https://shavezahmedsiddiqui.com/).

Use this as a conceptual reference and cite the repository and version if adapting the design. **No open-source license is granted by this README alone**; maintainers should choose an explicit license before inviting code reuse or contributions.

## Disclaimer

This material is for technical research and documentation. It does not constitute a financial product, security audit, banking license, legal determination, or investment recommendation. Product behavior, regulatory duties and custody classification depend on the actual deployment and contractual facts.
