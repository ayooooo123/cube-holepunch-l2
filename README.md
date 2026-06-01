# Cube Holepunch Bitcoin L2

A Cube-native Bitcoin execution-network specification that uses Holepunch for peer coordination and Autobase for decentralized pre-publication ordering of Cube entries, batches, and settlement artifacts.

> Design rule: **Holepunch distributes, Autobase coordinates, Cube executes, Bitcoin settles.**

## Status

This document is a protocol specification, not a claim that the current design is already a fully trustless Bitcoin rollup.

The credible path has three security profiles:

1. **Signet/testnet prototype:** prove Cube batch propagation, deterministic replay, watcher verification, and Bitcoin anchoring without custody risk.
2. **Federated mainnet network:** useful and buildable with capped custody, threshold operators, transparent accounting, watcher replay, and explicit trust assumptions.
3. **Trust-minimized Bitcoin L2:** requires an enforceable challenge, proof, or unilateral exit path on Bitcoin. This likely needs BitVM-style disputes, carefully constrained scripts, future Bitcoin covenant/opcode support, or another concrete settlement mechanism.

The spec is intentionally written to avoid fake rollup language. A root posted to Bitcoin is only a commitment. The system becomes a legitimate BTC L2 only when users can verify, challenge, or exit through Bitcoin-enforced rules under the advertised fault model.

## Cube Primer

Cube is not an EVM clone. It is a Bitcoin-oriented execution environment built around compact batch payloads, Bitcoin transaction outputs, and a custom script-like VM.

Relevant Cube concepts:

- **Entry:** a user or protocol action.
- **Move:** moves coins between accounts.
- **Call:** calls a contract.
- **Add/Sub:** adds or removes liquidity from the Engine.
- **Liftup:** imports one or more Bitcoin `Lift` outputs into Cube accounting.
- **Swapout:** exports account coins back into a bare Bitcoin transaction output.
- **Deploy:** deploys a Cube contract.
- **Config:** configures or reconfigures an account.
- **Payload:** compact encoded batch payload, using Airly Payload Encoding.
- **BatchContainer:** Bitcoin-oriented container for a batch height, payload bytes, and signed batch transaction.
- **BatchRecord:** executed batch metadata: batch height, txid, entries, payload, projector data, signatures, expired projectors, and related artifacts.
- **Projector:** Bitcoin output/artifact used by Cube batch construction and state progression.
- **Engine:** operating entity that builds/signs/coordinates batch execution under the current Cube model.
- **Watcher:** independent node that fetches batch data, replays Cube, checks roots/accounting, and alerts or challenges.

Cube’s execution model is a better starting point for Bitcoin than “EVM on Bitcoin” because it optimizes for Bitcoin’s constraints: blockspace, UTXOs, Taproot/Schnorr, batch publication, and compact data availability.

## Non-Goals

This design does **not** claim that:

- Holepunch provides consensus.
- Autobase provides Byzantine finality.
- Cube execution alone secures BTC without a settlement path.
- Engine-signed batches are automatically trustless.
- Bitcoin checkpointing is enough without exits, challenges, or proof verification.
- P2P replication is equivalent to Bitcoin data availability.
- Mainnet custody is safe before the bridge model is proven under adversarial tests.

## Architecture Summary

The system is organized around Cube’s native lifecycle.

```text
Wallet/Account
  -> Cube Entry
  -> Holepunch gossip
  -> Autobase proposal/order
  -> Cube batch construction
  -> BatchContainer / BatchRecord
  -> Bitcoin publication or anchoring
  -> Watcher replay
  -> settlement / withdrawal / challenge
```

The core design choice is that Autobase is a **pre-publication coordination layer**, not the final settlement chain.

Autobase can help decentralize mempool ordering, batch proposal, and operator coordination. The canonical settled state must still be derived from Cube batch artifacts and whatever Bitcoin can enforce under the active security profile.

## Layer Responsibilities

### 1. Wallet and Account Layer

Wallets create Cube-native entries, not generic rollup transactions.

Wallet responsibilities:

- maintain account key material
- encode Cube entries using the active payload rules
- sign entries with the required signature scheme
- submit entries over Holepunch
- track lifecycle state: gossiped, ordered, batched, Bitcoin-published, settled
- verify inclusion in a batch record before treating a transfer as durable
- refuse unsafe finality claims when data or settlement proofs are missing

### 2. Holepunch Coordination Layer

Holepunch provides peer discovery, encrypted transport, NAT traversal, and artifact replication.

Holepunch distributes:

- pending Cube entries
- Autobase writer feeds
- proposed batches
- BatchContainers
- BatchRecords
- payload blobs
- Projector, Liftup, and Swapout artifacts
- watcher attestations and fraud alerts
- snapshots and replay checkpoints

Holepunch messages are never trusted by transport identity alone. Every artifact must be content-addressed, signed where appropriate, and validated by Cube or the watcher rules.

### 3. Autobase Ordering Layer

Autobase is used as a decentralized coordination log for candidate Cube entries and batch proposals.

Autobase responsibilities:

- accept writes from an explicit sequencer/proposer set
- preserve causal references between proposals
- expose an eventually consistent pre-publication ordering
- help multiple independent operators converge on the same candidate batch range
- provide public observability into censorship, delay, and competing proposals

Autobase does **not** decide Cube validity. It does **not** make a withdrawal safe. It does **not** replace Bitcoin publication.

The protocol must distinguish four ordering states:

```text
Gossiped -> Autobase-ordered -> Cube-batched -> Bitcoin-settled
```

- **Gossiped:** peers have seen the entry; it may disappear.
- **Autobase-ordered:** a proposal log includes it; ordering may still converge or be superseded.
- **Cube-batched:** the entry appears in a Cube BatchContainer/BatchRecord and can be replayed.
- **Bitcoin-settled:** the relevant Bitcoin commitment, challenge window, proof policy, or withdrawal rule has matured.

### 4. Cube Execution Layer

Cube is the execution and accounting engine.

Cube responsibilities:

- decode Airly Payload Encoding
- execute Cube entries in batch order
- enforce account and contract rules
- validate Liftup and Swapout semantics
- track balances, storage, shadow allocations, and contract calls
- produce deterministic BatchRecords
- expose replayable state transitions for watchers
- maintain bridge/accounting invariants

Cube VM characteristics:

- stack-based, Bitcoin Script-like execution
- custom opcodes for control flow, stack manipulation, splicing, bitwise operations, arithmetic, hashing, secp operations, Schnorr/BIP340 checks, BLS checks, calls, storage, memory, balances, and transfers
- `Ops` budget model instead of EVM-style gas
- compact calldata and method encoding
- BLS aggregation strategy for signature compression

The spec should treat Cube as the canonical execution language, not as a placeholder for “some VM.”

### 5. Bitcoin Settlement Layer

Bitcoin provides the final settlement substrate.

Bitcoin is used for:

- Lift outputs and deposit/lift recognition
- batch transaction publication or anchoring
- payload/projector commitment
- Swapout settlement
- operator or signer accountability
- challenge/proof publication where supported
- emergency withdrawal paths where the settlement profile supports them

Bitcoin cannot cheaply verify arbitrary Cube execution today. Therefore each deployment must state exactly what Bitcoin enforces and what remains a social, federated, or watcher assumption.

## Cube-Native Object Model

### Cube Entry

A Cube Entry is the primitive unit of execution.

Entry kinds:

```text
Move
Call
Add
Sub
Liftup
Swapout
Deploy
Config
Nop
Fail
```

Required envelope fields for P2P/batch coordination:

```text
version
cube_chain_id
entry_kind
account_or_contract_ref
encoded_entry_payload
payload_hash
fee_or_ops_policy
expiry_height_or_deadline
signature_scheme
signature_or_aggregate_ref
```

Rules:

- Entries must be valid under Cube’s APE and entry decoding rules.
- Pending entries may be gossiped before inclusion.
- Only Cube-batched entries affect canonical state.
- Failed or invalid entries should be handled according to Cube’s assertion model: invalid entries must not silently mutate state.

### Autobase Batch Proposal

Autobase should carry proposals, not final state.

Fields:

```text
proposal_version
autobase_key
writer_key
writer_sequence
causal_refs
proposed_batch_height
entry_hashes
payload_manifest_ref
previous_cube_batch_ref
proposer_signature
```

Rules:

- Proposal order helps build batches but does not settle them.
- Multiple competing proposals are allowed; watchers should observe and score proposer behavior.
- A proposal with unavailable payload data is ineligible for batch construction.

### Cube BatchContainer

Cube’s BatchContainer should remain the batch publication primitive.

Conceptual fields:

```text
batch_height
new_payload_bytes
signed_batch_txn
```

Rules:

- Batch height must advance according to Cube rules.
- Payload bytes must decode under the active APE version.
- Signed batch transaction must bind payload/projector/lift/swapout artifacts according to Cube’s Bitcoin transaction rules.

### Cube BatchRecord

BatchRecord is the replay/watcher artifact.

Conceptual fields:

```text
batch_height
batch_txid
batch_timestamp
entries
aggregate_signature_or_signature_set
expired_projector_outpoints
new_payload
new_projector_optional
batch_container
```

Rules:

- Watchers must be able to reconstruct BatchRecord from public data plus replicated artifacts.
- BatchRecord must be sufficient to replay Cube state from the previous checkpoint/snapshot.
- BatchRecord hashes should be content-addressed and propagated over Holepunch.

### Watcher Receipt

A watcher receipt is not settlement by itself. It is an auditable claim that a watcher replayed a range.

Fields:

```text
watcher_key
from_batch_height
to_batch_height
batch_record_root
payload_manifest_root
pre_state_root
post_state_root
bridge_accounting_root
liftup_root
swapout_root
status
error_or_fraud_ref_optional
signature
```

Rules:

- Wallets may use watcher receipts as confidence signals.
- Settlement rules must not rely on a single watcher.
- Fraud alerts should include enough data to reproduce the failure.

## Airly Payload Encoding and Data Availability

Cube’s APE is central to the design. The protocol should preserve its advantages instead of wrapping everything in generic JSON/RLP objects.

APE goals:

- bit-level transaction and value encoding
- common value lookup
- rank-based account/contract indexing
- non-prefixed calldata where types are known
- compact call method selectors
- signature aggregation
- nonceless entry encoding where valid under Cube’s external chaining model
- ops-budget encoding instead of gas-price/gas-limit overhead
- assertion-based inclusion to avoid carrying failed transactions as state-changing records

Data availability rules:

- Every batch must have a payload manifest listing content hashes for all required replay data.
- Batch payloads should be pinned by independent Holepunch peers before a batch is considered safe for wallet finality.
- Watchers must fetch and replay payloads, not merely observe batch headers.
- If a payload is unavailable, the batch is unsafe even if some commitment exists.
- If full data is on Bitcoin, state that explicitly and measure the fee impact. If data is off-chain, state the DA assumption directly.

## Liftup and Swapout Semantics

Liftup and Swapout are the heart of the BTC bridge model.

### Liftup

Liftup imports Bitcoin value into Cube.

A Liftup must define:

- Bitcoin outpoint being lifted
- script template or recognition rule
- amount
- account recipient
- engine/projector relationship
- confirmation policy
- replay protection
- batch height at which the lift is recognized

Open enforcement question:

```text
Can a user prove a valid Liftup independently if the Engine refuses to include it?
```

If not, Liftup is operator-mediated and the trust model must say so.

### Swapout

Swapout exports Cube value back to Bitcoin.

A Swapout must define:

- Cube account authorization
- amount
- destination Bitcoin script
- batch height and record reference
- withdrawal queue position, if any
- maturity/challenge policy
- signer or script path used to release funds

Open enforcement question:

```text
Can a user force a Swapout on Bitcoin if the Engine, sequencers, or bridge signers stop cooperating?
```

If not, the system is federated/custodial at the exit layer even if execution is reproducible.

## Bridge Accounting Invariants

At every replayed batch height, watchers must check:

```text
locked_btc >= cube_btc_claims + pending_swapouts - burned_or_slashed_amounts
```

Additional invariants:

- one Bitcoin outpoint can be lifted at most once
- one Cube balance claim can be swapped out at most once
- expired projectors cannot be reused
- aggregate signatures must cover the exact entries or batch payload they claim to authorize
- batch height must be monotonic
- payload/projector references must match the signed batch transaction
- rank-based account/contract indexes must be deterministic from replayed history

Any invariant failure makes the batch invalid under Cube replay, even if the batch was gossiped or proposed through Autobase.

## Security Profiles

### Profile A: Signet/Testnet Prototype

Purpose: prove engineering correctness with no real custody risk.

Requirements:

- run Cube on signet or regtest
- publish or anchor sample batches
- propagate entries and BatchRecords over Holepunch
- use Autobase for proposal ordering
- run independent watcher replay
- include invalid-batch fixtures
- demonstrate Liftup and Swapout on signet

Success criteria:

- a fresh node can sync from peers
- a watcher can replay from genesis or snapshot
- invalid payloads are rejected deterministically
- missing payloads are detected
- a signet Liftup and Swapout complete through the documented flow

### Profile B: Federated Mainnet Candidate

Purpose: useful BTC-backed execution with explicit trust assumptions.

Requirements:

- threshold-controlled Engine/signer set
- public operator keys
- custody caps
- withdrawal delays
- independent watchers
- public BatchRecords and payload manifests
- emergency pause rules
- user-facing warning that exits require federation cooperation

Security assumption:

```text
At least the threshold signer honesty assumption holds, and watchers can publicly detect/account for invalid behavior.
```

This can be a real product, but should be marketed honestly as federated until exit enforcement improves.

### Profile C: Optimistic Bitcoin L2

Purpose: reduce operator trust through challengeable invalid execution.

Requirements:

- precise fraud claim format
- deterministic Cube execution trace format
- dispute game that narrows disagreement to a verifiable step or bridge-accounting assertion
- Bitcoin transaction/script flow that can enforce the challenge outcome
- challenge window long enough for data retrieval and mempool fee spikes
- one-honest-watcher assumption stated directly

Security assumption:

```text
At least one honest watcher obtains the data and can complete the challenge on Bitcoin within the challenge window.
```

This profile is research-heavy. It should not be claimed until the exact Bitcoin enforcement path exists.

### Profile D: Validity/BitVM Settlement

Purpose: strongest long-term trust minimization.

Requirements:

- Cube execution compiled to a provable circuit/VM or BitVM-compatible dispute program
- proof or claim commitment in Bitcoin-visible artifacts
- soundness assumptions documented
- fallback challenge path for invalid proof claims
- proof witness/data availability policy

Security assumption:

```text
Proof/dispute soundness plus data availability.
```

This is the path to a stronger “legit L2” claim, but it is not the MVP.

## Watcher Design

Watchers are first-class actors.

Watcher responsibilities:

- sync Autobase proposals
- fetch Cube BatchContainers and BatchRecords
- fetch payload/projector/lift/swapout artifacts over Holepunch
- replay Cube entries from genesis or snapshot
- verify bridge accounting invariants
- compare roots and batch txids
- detect unavailable data
- emit signed receipts and fraud alerts
- optionally submit challenges under optimistic/validity profiles

Watcher CLI MVP commands:

```text
watcher sync --from <batch_height>
watcher replay --from <snapshot> --to <batch_height>
watcher verify-batch <batch_height>
watcher verify-liftup <outpoint>
watcher verify-swapout <batch_height>:<index>
watcher emit-receipt <from> <to>
watcher fraud-report <batch_height>
```

## Network Protocol Sketch

Holepunch topics should be keyed by network and artifact type.

```text
cube/<chain_id>/entries
cube/<chain_id>/autobase/writers
cube/<chain_id>/batchcontainers
cube/<chain_id>/batchrecords
cube/<chain_id>/payloads
cube/<chain_id>/projectors
cube/<chain_id>/liftups
cube/<chain_id>/swapouts
cube/<chain_id>/watchers
cube/<chain_id>/fraud
```

Each artifact should have:

```text
artifact_type
version
chain_id
content_hash
producer_key
signature_optional
body_or_body_ref
```

Peers should reject oversized, malformed, wrong-chain, unsigned-when-required, or payload-missing artifacts before relaying them further.

## MVP Build Plan

The first credible build should be narrow.

### Phase 0: Spec/Test Vector Alignment

- Define Cube chain ID and network profile.
- Define canonical hashes for Entry, BatchProposal, BatchContainerRef, BatchRecordRef, WatcherReceipt.
- Add fixtures for Move, Liftup, Swapout, Deploy, and invalid entries.
- Document what fields come directly from Cube and what fields are added by the Holepunch/Autobase coordination layer.

### Phase 1: Holepunch Artifact Bus

- Build a Holepunch peer that gossips Cube entries and batch artifacts.
- Content-address payloads and BatchRecords.
- Add pinning for replay-critical data.
- Add peer diagnostics for missing payloads and replication count.

### Phase 2: Autobase Batch Proposal Log

- Create an Autobase with a fixed writer set.
- Append Cube entry hashes and batch proposals.
- Track pending vs stable proposal order.
- Expose a read model for candidate batch construction.
- Log censorship and competing proposal evidence.

### Phase 3: Cube Replay Watcher

- Import Cube BatchContainers and BatchRecords.
- Replay batches from genesis or snapshot.
- Verify bridge/accounting invariants.
- Emit signed watcher receipts.
- Generate fraud fixtures for invalid batch data.

### Phase 4: Signet Liftup/Swapout Demo

- Run Bitcoin signet node or use configured signet RPC.
- Execute Liftup into Cube.
- Move value inside Cube.
- Execute Swapout to signet output.
- Publish a full transcript: Bitcoin txids, Cube batch heights, payload hashes, watcher receipts.

### Phase 5: Federated Mainnet Readiness Review

- Define custody cap.
- Define threshold signer policy.
- Define emergency halt and restart rules.
- Define withdrawal delay.
- Require at least three independent watcher operators.
- Run adversarial tests before any real custody.

## Required Proof Before “Legit L2” Claim

Do not call the system a trust-minimized Bitcoin L2 until at least one of these is true:

1. **Unilateral exit exists:** users can recover funds through Bitcoin without Engine cooperation.
2. **Fraud challenge exists:** invalid Cube batches can be challenged and blocked on Bitcoin.
3. **Validity settlement exists:** Bitcoin or a Bitcoin-enforceable mechanism accepts only valid Cube state transitions.
4. **Federation honesty is the explicit product:** the system is marketed as a federated BTC execution network, not as trustless.

The strongest honest claim before that point is:

```text
Cube Holepunch is a Bitcoin-native federated execution network with decentralized P2P coordination, deterministic replay, and a path toward stronger Bitcoin-enforced settlement.
```

## Open Questions

1. What exactly can the Cube Engine do unilaterally today?
2. Which Cube artifacts are always on Bitcoin, and which are off-chain DA?
3. Can a Liftup be forced or only recognized by Engine inclusion?
4. Can a Swapout be forced if the Engine halts?
5. How are Projectors created, expired, and prevented from reuse?
6. What is the exact meaning of batch finality under Cube today?
7. How does Cube’s nonceless model prevent replay across Autobase proposal forks?
8. How are Autobase writers admitted, removed, and punished?
9. Does BLS aggregation introduce rogue-key or registration assumptions?
10. What is the minimum data a watcher needs to replay from a snapshot?
11. Can Cube execution be compiled into a BitVM-compatible fraud game?
12. What is the smallest useful signet demo that proves Liftup -> Move -> Swapout?

## Bottom Line

The strongest version of this idea is not “a generic L2 that happens to use Holepunch.” It is a Cube-native Bitcoin execution network:

```text
Cube gives the Bitcoin-shaped execution and batch format.
Holepunch gives the P2P fabric for entries, payloads, batch records, and watcher artifacts.
Autobase gives decentralized pre-publication ordering and proposer accountability.
Bitcoin gives final settlement only to the extent that Liftup, Swapout, challenge, proof, or federation rules can actually enforce it.
```

This is buildable as a serious signet prototype and probably as a useful federated BTC-backed network. It becomes a legitimate trust-minimized Bitcoin L2 only when the exit/challenge/proof path is concrete enough that users are protected from a dishonest or offline Engine.
