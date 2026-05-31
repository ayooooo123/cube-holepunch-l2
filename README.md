# Cube Holepunch L2

A technical specification and whitepaper for a Bitcoin L2 architecture that uses Autobase as a decentralized sequencer, with Holepunch for coordination and Cube for enforcement.

## Abstract

Cube Holepunch L2 is a Bitcoin-native execution and settlement design centered on a simple separation of concerns:

- Holepunch coordinates participants, data propagation, discovery, and peer connectivity.
- Autobase provides a decentralized sequencer for ordering L2 intents and producing a shared transaction log.
- Cube enforces validity by executing the sequenced state transitions, validating proofs and policy, and determining the canonical L2 state.
- Bitcoin anchors final settlement through periodic commitments and dispute resolution.

The primary goal is to minimize trust in any single operator while preserving a practical path to high-throughput L2 execution. Coordination is treated as a networking problem, sequencing as a log replication problem, and enforcement as a deterministic state transition problem.

## Design Philosophy

This project follows a single sentence design rule:

> Holepunch for coordination, Cube for enforcement.

That means:

1. Coordination should be peer-to-peer, resilient, and easy to join.
2. Sequencing should be decentralized and observable.
3. Enforcement should be deterministic, reproducible, and minimal in assumptions.
4. Bitcoin should be the ultimate settlement and security anchor.

The design intentionally avoids conflating coordination, ordering, and validity. Each layer can be independently replaced or upgraded as long as its interface remains stable.

## System Overview

The system consists of five layers:

1. Participant layer
   - Wallets, users, operators, and validators.
   - Users submit intents, transfers, or programmatic calls.

2. Coordination layer
   - Holepunch-based peer discovery and messaging.
   - Used for gossip, routing, membership hints, and availability.

3. Sequencing layer
   - Autobase maintains a decentralized append-only log of L2 events.
   - Events are ordered into a canonical view for execution.

4. Enforcement layer
   - Cube ingests the ordered log and runs deterministic validation.
   - Invalid transitions are rejected, quarantined, or provably excluded.

5. Settlement layer
   - Bitcoin receives checkpoints, fraud proofs, or validity commitments.
   - Finalized L2 state is anchored to L1.

## Core Architecture

### 1. Coordination with Holepunch

Holepunch is used to connect participants without requiring a central coordinator. In this model, Holepunch provides:

- peer discovery
- transport establishment
- live gossip of pending intents and log heads
- membership hints for validators and sequencers
- content-addressed distribution of log segments and proofs

Holepunch does not decide validity. It exists to make the network easy to find, hard to censor, and efficient to synchronize.

### 2. Sequencing with Autobase

Autobase acts as the decentralized sequencer. Its job is to produce a consistent ordering of L2 records across many writers. The output is a shared log that can be consumed by all enforcement nodes.

In this design, Autobase is responsible for:

- collecting candidate L2 events from multiple writers
- ordering those events into a total or near-total view for execution
- preserving causality where required
- exposing a canonical feed for Cube
- enabling concurrent writes without a single centralized sequencer

Autobase is not the validity engine. It orders what is proposed, not what is allowed.

### 3. Enforcement with Cube

Cube is the enforcement engine. It consumes Autobase output and determines the valid L2 state transition sequence.

Cube responsibilities:

- verify transaction syntax and signatures
- enforce account and balance rules
- execute L2 programs or contract logic deterministically
- validate bridge deposits and withdrawals
- check replay protection and nonce rules
- ensure state commitments match execution output
- produce proofs or attestations for checkpointing

If Autobase is the shared log, Cube is the court that decides what counts.

## Data Model

The system distinguishes between three classes of data:

### Intent

An intent is a user-signed statement of desired action.

Examples:
- transfer tokens
- withdraw to Bitcoin
- interact with a contract
- register a state update

Intents are lightweight and can be broadcast before final inclusion.

### Sequenced Event

A sequenced event is an intent that has been ordered by Autobase.

It includes:
- log position
- author identity
- timestamp or logical clock information
- parent or causal references, if any
- payload hash

### State Transition

A state transition is the result of Cube executing a sequenced event against the current state.

It includes:
- pre-state root
- post-state root
- validity result
- any generated outputs
- proof or attestation references

## Execution Model

Cube executes the log in order and applies deterministic rules.

The execution pipeline is:

1. Receive sequenced event from Autobase.
2. Validate envelope, signatures, and anti-replay conditions.
3. Load prior state from the local state store.
4. Execute event logic.
5. Compute new state root.
6. Emit validity result and derived outputs.
7. Persist execution artifacts.

This model allows many nodes to independently derive the same result from the same log.

## Transaction Lifecycle

### Step 1: Construction

A user creates an intent locally using wallet software.

### Step 2: Coordination

The intent is gossiped over Holepunch to peers that participate in sequencing or monitoring.

### Step 3: Sequencing

Autobase orders the intent relative to other pending records.

### Step 4: Enforcement

Cube validates and executes the ordered record.

### Step 5: Commitment

A checkpoint or proof is periodically posted to Bitcoin.

### Step 6: Finality

Once the L1 commitment matures under the chosen policy, the L2 state is considered settled.

## Bitcoin Settlement Model

Bitcoin serves as the root of final settlement, but not every L2 event requires an immediate L1 write.

The protocol can support several settlement modes:

- periodic checkpointing of state roots
- batched withdrawals
- fraud-proof based challenge windows
- validity proof anchoring, if available
- hybrid checkpoints combining attestation and delayed settlement

The exact settlement policy can evolve, but the key invariant remains: Bitcoin is the final arbiter of canonical exits and long-term settlement.

## Consensus and Trust Assumptions

The architecture is intentionally modular.

### Coordination assumptions

Holepunch is assumed to provide connectivity but not consensus.

### Sequencing assumptions

Autobase provides decentralized ordering semantics, but the protocol should not assume perfect honesty among writers.

### Enforcement assumptions

Cube is trusted only to the extent that its code is reproducible and independently verifiable by all participants.

### Settlement assumptions

Bitcoin is the only layer treated as maximally trusted for final settlement.

The model reduces trust by distributing responsibilities:

- no single coordinator controls the network
- no single sequencer defines validity
- no single execution node defines the state
- no single bridge operator can mint canonical value unilaterally

## Validity Rules

Cube must enforce the following classes of invariants:

- signature authenticity
- state transition determinism
- account balance conservation
- nonce and replay constraints
- bridge mint/burn accounting
- withdrawal authorization
- bounded execution rules
- protocol upgrade authorization

Additional application-specific rules may be layered on top.

## Invalid Data Handling

Sequencing and validity are distinct. Not every ordered event is valid.

Cube should support:

- rejection of invalid events
- quarantine of malformed log entries
- audit trails for rejected transitions
- deterministic re-execution from genesis or checkpoint

Invalid events remain observable in the log for transparency, even if they do not affect canonical state.

## State Representation

The L2 state may be represented as:

- account-based balances and nonces
- UTXO-derived commitments
- Merkleized contract storage
- a hybrid model combining accounts and objects

For a Bitcoin L2, a Merkle commitment scheme is strongly preferred so the state can be compactly verified, checkpointed, and challenged.

Recommended state outputs:

- state root
- execution root
- withdrawal queue root
- validator set root
- bridge accounting root

## Bridge Design

A Bitcoin L2 needs careful deposit and withdrawal semantics.

### Deposits

Deposits are recognized when Bitcoin transactions or scripts satisfy the bridge’s deposit conditions. Cube credits the corresponding L2 account only after the deposit is validated according to bridge policy.

### Withdrawals

Withdrawals require:

- valid L2 authorization
- spent balance or escrow removal in Cube
- a settlement path to Bitcoin
- challenge or delay logic if the bridge is optimistic

### Conservation rule

At all times, total outstanding L2 claims must reconcile with the committed bridge accounting model.

## Fault Model

The system is designed to tolerate:

- node failures
- peer churn
- sequencer disagreement
- delayed log propagation
- partial partitions
- malicious invalid transactions
- equivocation by some writers

The system does not assume that every peer is online or honest.

Recovery is achieved by replaying the log and re-validating state from deterministic rules.

## Security Properties

Target properties include:

- data availability through replicated peer gossip
- ordering transparency through a shared sequenced log
- validity enforcement through deterministic execution
- censorship resistance through distributed coordination
- settlement safety through Bitcoin anchoring

Potential attack surfaces include:

- log withholding
- equivocation among writers
- invalid state proposals
- bridge abuse
- metadata correlation
- delayed checkpoint publication

Mitigations should be implemented at each layer rather than by a single monolithic mechanism.

## Performance Goals

The architecture aims for:

- low-latency gossip of user intents
- high-throughput sequencing via Autobase
- parallelized validation across Cube nodes
- batched Bitcoin settlement to reduce L1 overhead
- efficient incremental state updates

Performance is improved by separating fast-path coordination from slow-path settlement.

## Upgradeability

Protocol upgrades should be explicit and versioned.

Recommended upgrade surfaces:

- log format versioning
- execution rule versioning
- bridge policy versioning
- checkpoint policy versioning
- consensus metadata versioning

Upgrades should be stateful but backward-readable wherever possible.

## Reference Architecture

### Node types

- Wallet node: creates and signs intents.
- Peer node: relays and stores log fragments.
- Sequencer participant: contributes to Autobase ordering.
- Cube validator: executes and enforces transitions.
- Watcher: independently monitors state and settlement.
- Bridge relayer: submits settlement artifacts to Bitcoin.

### Suggested deployment model

- Many small coordination nodes.
- Multiple independent sequencer participants.
- Redundant Cube validators with identical logic.
- At least one watcher per independent operator.

## Minimal Protocol Interfaces

### Intent envelope

- sender
- signature
- nonce
- payload type
- payload body
- fee or priority hints

### Sequenced record

- log index
- writer identity
- causal references
- payload hash
- network metadata

### Execution receipt

- pre-state root
- post-state root
- validity flag
- emitted events
- proof reference

### Settlement artifact

- checkpoint height
- state root
- inclusion evidence
- challenge parameters
- Bitcoin transaction reference

## Roadmap

Phase 1
- Define the intent envelope and execution receipts.
- Implement Holepunch coordination and log propagation.
- Establish Autobase sequencing semantics.
- Build Cube execution for core transfers.

Phase 2
- Add deposits, withdrawals, and bridge accounting.
- Add checkpointing to Bitcoin.
- Implement watcher and challenge tooling.

Phase 3
- Expand contract support.
- Improve proof generation.
- Optimize state storage and replay.

Phase 4
- Harden governance and upgrade mechanisms.
- Formalize interoperability with wallets and external systems.

## Open Questions

- Should the sequencing layer produce a strict total order or a causally ordered canonical merge?
- What is the best balance between optimistic and validity-based settlement?
- How should the bridge represent partial finality?
- Which execution model best fits Bitcoin-native constraints: accounts, UTXO derivatives, or hybrids?
- What proof system, if any, should Cube eventually support?

## Conclusion

Cube Holepunch L2 proposes a clean separation of coordination, sequencing, and enforcement.

Holepunch makes the network reachable.
Autobase makes the event stream decentralized.
Cube makes the state transition enforceable.
Bitcoin makes the system settle.

This separation is the core of the architecture: Holepunch for coordination, Cube for enforcement.
