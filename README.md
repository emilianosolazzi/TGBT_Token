TEMPORAL GRADIENT
BEACON TOKEN
TGBT

Technical Whitepaper  —  Version 1.0  —  April 2026
Bitcoin-anchored, proof-of-work mined ERC-20 on Arbitrum

	Anyone can mine Bitcoin. Nobody can mine Bitcoin anymore. TGBT reopens that window — same philosophy, same hard cap, same proof-of-work soul — but accessible today, on any CPU, before the network gets competitive.

 
Abstract

The Temporal Gradient Beacon Token (TGBT) is an immutable-cap ERC-20 token deployed on Arbitrum with a hard supply ceiling of 2,000,000,000 tokens. TGBT is earned exclusively through an on-chain proof-of-work mining protocol that anchors entropy to real Bitcoin block data, creating a cryptographically verifiable randomness beacon with economic incentives aligned to genuine computational contribution.

TGBT introduces a Bitcoin-inspired ossification model: governance retains limited, time-bounded authority to configure authorized modules during a bootstrap phase. Once the module set is stable, a single irreversible transaction — ossify() — permanently revokes all governance power, making the token contract and its emission schedule as immutable as Bitcoin itself. No admin mint. No pause. No upgrade proxy. No going back.

Bitcoin made computation prove economic security. Temporal Gradient makes computation prove operational reality.
 
1. Introduction

Most ERC-20 tokens launched today carry hidden administrative risk: owners who can mint at will, proxies that can be upgraded silently, or multisigs that can pause transfers indefinitely. TGBT is designed from first principles to eliminate these trust assumptions without sacrificing a sophisticated tokenomics model.

The core insight is borrowed from Bitcoin: a sufficiently well-designed protocol does not need ongoing human governance once it reaches steady state. TGBT's module architecture allows iterative improvement during a public bootstrap window, after which ossify() permanently removes all administrative authority in a single, auditable on-chain transaction.

Simultaneously, TGBT's emission schedule is not arbitrary. Every token is earned through a commit-reveal proof-of-work protocol that incorporates real Bitcoin block merkle roots as an external entropy source — making the randomness beacon resistant to manipulation by any single party, including the protocol deployer.

	Bitcoin's proof-of-work concentrated into industrial mining because global monetary settlement justifies that energy expenditure. TGBT solves a different problem — device liveness, entropy generation, and provenance anchoring — where breadth of participation is more valuable than concentrated hash power. A thousand laptops across a thousand locations is more valuable to this network than one warehouse.
 
2. Token Design

2.1 Core Parameters

Parameter	Value
Token Name	Temporal Gradient Beacon Token
Ticker	TGBT
Network	Arbitrum (L2)
Standard	ERC-20 (OpenZeppelin — no proxy, no upgrade)
Hard Cap	2,000,000,000 TGBT
Mining Allocation	1,900,000,000 TGBT (95%)
Stale Block Allocation	75,000,000 TGBT (3.75%)
Cap Headroom	25,000,000 TGBT (1.25%)
Admin Mint	None — ever
Upgrade Proxy	None
Pause Function	Bootstrap only; permanently disableable via ossify()

2.2 Authorization Model
TGBT uses a two-phase authorization model inspired by Bitcoin's development lifecycle:

•	Bootstrap Phase: The immutable governance address can grant and revoke module authorizations. Only authorized modules can call mint() or recordStamp(). This phase exists to allow safe iterative deployment of the module set.
•	Ossification Phase: lockPermissions() permanently freezes all authorizations. After this call, the governance address has zero on-chain power. This action is irreversible by design.

The authorization check is enforced at the token contract level. No authorized module can exceed the hard cap. No entity — including the deployer — can mint tokens outside of the authorized module set.

2.3 The Hard Cap Guarantee
The 2,000,000,000 TGBT ceiling is enforced by the token contract itself, independent of any module. Every call to mint() checks totalSupply() + amount > MAX_SUPPLY and reverts with CapExceeded if the cap would be breached. This check cannot be bypassed or modified after deployment because the token contract has no upgrade mechanism. Even a compromised module cannot over-mint.
 
3. Mining Protocol

3.1 Overview
TGBT is earned through two complementary mining paths: direct commit-reveal mining via MiningModule, and batch epoch mining via BatchMiningModule. Both paths are protected by the same hard cap and tokenomics enforcement. The commit-reveal path rewards individual solutions in real time; the batch path accumulates solutions into Merkle epochs and finalizes them after a challenge window.

3.2 Commit-Reveal Mining (MiningModule)
The commit-reveal protocol requires two on-chain transactions per mining attempt, preventing front-running and ensuring each submitted solution represents genuine pre-committed computational work.

Commit Phase
A miner computes a candidate solution off-chain, then submits a commitment — a hash of their solution parameters — via submitMiningCommitment(). The commitment is signed using EIP-712 typed data, binding it to the miner's address, pool ID, nonce, and a deadline. This prevents commitment spoofing and nonce replay. The commitment must age at least minCommitmentAge blocks (default: 2) before it can be revealed, preventing same-block front-running. Commitments expire after maxCommitmentAge blocks (default: 500).

Reveal Phase
The miner calls revealMiningCommitment() with their original inputs, including a temporalSeed — an 8-byte BOM-prefixed Unix timestamp — along with a previousOutput drawn from the protocol's 32-slot circular output history, a nonce, a signature, and a secret value. The contract verifies:

•	The commitment hash matches the revealed inputs exactly.
•	The temporal seed is within an acceptable time window (not older than 30 days, not more than 15 minutes in the future, not drifting more than 1 hour from block.timestamp).
•	The previousOutput exists in the protocol's rolling 32-slot output history — anchoring each solution to real on-chain state.
•	The ECDSA signature over the entropy hash recovers to the miner's address.
•	The resulting hmacOutput meets the pool's target difficulty.
•	The hmacOutput has not been used before (duplicate prevention via usedOutputs mapping).

3.3 Batch Epoch Mining (BatchMiningModule)
The batch path allows miners to accumulate multiple solutions off-chain, build a Merkle tree, and commit the root on-chain in a single transaction. This is more gas-efficient for continuous miners and creates an auditable epoch structure for the randomness beacon.

Constant	Value	Meaning
EPOCH_COOLDOWN_BLOCKS	50 L1 blocks	Min blocks between commits from same operator
CHALLENGE_WINDOW	28,800 L1 blocks	~96 hours before epoch can be finalized
MAX_LEAVES_PER_EPOCH	10,000	Maximum solutions per epoch
REWARD_PER_SOLUTION	1.375 TGBT	Minted per leaf on finalization

The lifecycle proceeds as follows:

1.	Miner accumulates accepted solutions in local state (epoch-state.json).
2.	When enough solutions are ready, epoch-builder commits the Merkle root on-chain via commitEpochRoot() with an EIP-712 signature.
3.	A 96-hour challenge window opens. Anyone can dispute a fraudulent root during this period.
4.	After the window elapses, the operator finalizes the epoch via finalizeEpoch(), minting leafCount × 1.375 TGBT.
5.	Optionally, a storage attestation hash (IPFS or other storage proof) is recorded on-chain via recordStorageAttestation().

The challenge window is a fraud-prevention mechanism — it gives the network time to dispute any incorrect epoch root before tokens are minted. Epochs can also be finalized manually for any epoch whose challenge window has elapsed, providing operational resilience independent of the automated epoch-builder.

3.4 Entropy Hash Construction
The mining output is computed using an iterative entropy hash — three rounds of keccak256 with a 7-bit rotation between rounds. This construction is bounded, deterministic, and produces outputs across the full 256-bit space, making difficulty targeting precise and manipulation-resistant.

The entropy inputs are: the miner's ECDSA signature, an entropy hash committing the previousOutput, temporalSeed, nonce, miner address, and secret value, and the secret value itself. No single party controls all inputs simultaneously — the entropy is genuinely multi-party.

3.5 Mining Pools
The protocol supports multiple mining pools, each with an independent target difficulty and emission bucket. Pools are created by governance before ossification and are immutable after creation — consistent with Bitcoin's philosophy that consensus rules should not change arbitrarily. Each pool tracks its own totalMined counter, and rewards are capped against both the global mining allocation and the individual pool emission bucket.
 
4. Bitcoin Temporal Anchoring

4.1 The Stamp System
TGBT's randomness beacon is anchored to real Bitcoin block data through the Stamp system. Authorized modules can call recordStamp() on the token contract, recording a cryptographic commitment to a specific Bitcoin transaction — identified by its transaction hash, output index (vout), block number, and the block's merkle root — alongside the Ethereum address of the contributing miner.

Each epoch can receive exactly one stamp. Once recorded, stamps are immutable and queryable via getEpochStamp(). The stamp system creates a public, on-chain audit trail linking TGBT epoch boundaries to specific Bitcoin blocks.

4.2 Bitcoin Inclusion Proofs
The recordStamp() function accepts an optional btcInclusionProof parameter — a byte array representing an SPV (Simplified Payment Verification) merkle proof that the recorded transaction is included in the stated Bitcoin block. When provided, the proof is stored as a keccak256 digest on-chain, enabling future on-chain SPV verification without storing the full proof bytes in contract storage.

The presence or absence of an inclusion proof is recorded in the StampRecorded event via the hasInclusionProof field, allowing off-chain verifiers to determine whether any given epoch anchor is fully SPV-verified or relies on the authorized module's attestation.

4.3 Stale Block Entropy Harvesting
Bitcoin occasionally produces stale blocks — valid blocks with real proof-of-work that lose the chain-tip race to a competing block at the same height. From Bitcoin's perspective this work is wasted. From Temporal Gradient's perspective, it is an exceptionally high-quality entropy source: unpredictable, unforgeable, and backed by real SHA-256 work at Bitcoin-level difficulty.

The stale block harvesting sidecar monitors Bitcoin chain tips for forks, detects orphaned blocks, extracts entropy from all 80 bytes of the block header, builds a StaleWorkProof, and submits it on-chain to the StaleBlockOracle contract. Valid proofs trigger a TGBT reward from the dedicated 75M TGBT stale block allocation — separate from the main mining budget.

	Temporal Gradient is, to our knowledge, the first system to systematically harvest the entropy embedded in Bitcoin's orphaned proof-of-work and anchor it to a separate chain. Every stale block is a gift from Bitcoin's competition — real, scarce, unmanipulable entropy that no purely on-chain system can replicate.

4.4 Dead-UTXO Anchoring
The system also extracts provenance value from permanently unspendable Bitcoin outputs — OP_RETURN outputs, spent outputs, dust, and known burn addresses. Each dead UTXO is converted into a canonical anchor record with a deterministic anchor_id computed as sha256(utxo_id ‖ data_hash ‖ merkle_root ‖ storage_reference ‖ created_at). This creates a verifiable bridge between a real Bitcoin output, a document or data digest, off-chain storage, and on-chain verification — without minting TGBT. Provenance has commercial value independent of token rewards.

4.5 Why Bitcoin Anchoring Matters
Bitcoin block production is controlled by global proof-of-work hash power distributed across thousands of independent mining operations. No single entity controls the contents of a Bitcoin block's merkle root. By incorporating Bitcoin merkle roots and stale block hashes as entropy sources, TGBT's randomness beacon inherits Bitcoin's manipulation resistance — a property that purely on-chain Ethereum randomness cannot achieve. The two entropy streams (internal mining and external Bitcoin harvesting) are independent, making the combined output strictly stronger than either source alone.
 
5. Emission Schedule

5.1 Epoch-Based Halving
TGBT's emission schedule is managed by the TokenomicsLib library, which implements a block-number-anchored epoch system. The schedule is initialized once and derives all future state deterministically from that initialization — no storage mutation is required to read the current reward, preventing manipulation through delayed state updates.

Rather than a hard halving (reward cut exactly in half), TGBT uses a 65/100 reduction per halving interval — a gentler curve that extends the productive mining lifecycle and avoids the sharp reward cliffs that can destabilize miner economics in hard-halving models.

5.2 Arbitrum Block Timing
The emission schedule uses Arbitrum L2 block numbers. Arbitrum produces blocks approximately every 0.25 seconds, meaning 1,000,000 blocks represents roughly 2.9 days and the maximum supported halving interval of 630,720,000 blocks represents approximately 5 years. Note: the BatchMiningModule challenge window uses L1 Ethereum block numbers (via Arbitrum's ArbSys), where 28,800 blocks ≈ 96 hours at a 12-second L1 block time. Both block number semantics are used in the system and are explicitly documented in the contract constants.

5.3 Bonus Rewards
Miners who produce solutions significantly exceeding the pool's target difficulty are eligible for a bonus multiplier. The default configuration applies a 1.25x multiplier when a solution's effective difficulty exceeds twice the pool target. The threshold and multiplier are set at module initialization and remain immutable thereafter, ensuring miners can predict exceptional reward conditions in advance.

Emission Parameter	Value
Reduction per halving	35% (65/100 multiplier — gentler than Bitcoin's 50%)
Minimum reward floor	1,000,000 wei (10⁻¹² TGBT)
Max halving reduction rounds	100 (overflow guard)
Epoch overflow guard	uint64 max epochs
Default bonus threshold	2× pool difficulty
Default bonus multiplier	1.25× base reward
Maximum bonus multiplier	5× base reward (500 bps cap)
 
6. Rate Limiting

All mining operations are protected by the RateLimitModule, which implements both per-user token bucket rate limiting and a global sliding window. Commit submissions cost 1 rate-limit token; reveal submissions cost 2. Users who exceed their individual capacity (default: 60 operations per refill window) are rejected without affecting other participants.

The rate limit configuration is set once at module initialization and is immutable thereafter. To change rate parameters, governance must deploy a new RateLimitModule and register it on Core — a transparent, auditable action that becomes impossible after ossification.
 
7. Ossification and Decentralization

7.1 The ossify() Function
The TemporalGradientCore contract contains a single function, ossify(), that permanently and atomically eliminates all governance authority in one transaction. When called, ossify():

•	Locks the module registry — no new modules can be registered or removed, ever.
•	Permanently disables the pause function — the system can never be paused again.
•	Revokes the GOVERNANCE_ROLE from all holders.
•	Revokes the DEFAULT_ADMIN_ROLE from all holders.
•	Renounces contract ownership.

Before ossification can proceed, the contract enforces a strict topology check: exactly one address holds each governance role, that address is the caller, the system is not currently paused, and at least one module is registered. This prevents accidental multi-address lockout and ensures ossification happens from a known clean state. The check is not advisory — it reverts if any condition is not met.

	Unlike tokens that promise decentralization without enforcement, TGBT's ossification is a single on-chain transaction, publicly observable and permanently auditable. Once ossify() is called, no human — including the deployer — has any administrative power over the protocol. This is not a marketing claim. It is enforced in Solidity.

7.2 Progressive Decentralization Timeline
TGBT follows a deliberate progressive decentralization model with three explicit phases:

•	Phase 1 — Bootstrap: Governance deploys and configures modules, sets pool parameters, and grants TGBT authorizations. The pause function is available as a safety valve. This is the current phase.
•	Phase 2 — Stabilization: Modules are tested in production. The community observes emission behavior and entropy quality. No parameter changes are made. Mining history and epoch records accumulate publicly.
•	Phase 3 — Ossification: Once the module set is stable and the community is satisfied, ossify() is called. From this point forward, no human has any administrative power over the protocol.

After ossification, TGBT's behavior is determined entirely by immutable on-chain code and the competitive dynamics of its mining participants — analogous to Bitcoin after its genesis block.
 
8. Security Considerations

8.1 Front-Running Prevention
The two-phase commit-reveal design ensures that no observer can profitably copy a mining solution after seeing it submitted. The commitment binds the solution hash on-chain before the solution is revealed. The minimum commitment age requirement (default: 2 blocks) ensures the committed hash cannot be front-run in the same block.

8.2 Duplicate Output Prevention
Every valid mining output is recorded in the usedOutputs mapping. Any attempt to reuse a previously accepted output is rejected with OutputAlreadyUsed. This prevents replay attacks and ensures each unit of computational work produces at most one reward.

8.3 Temporal Seed Validation
The 8-byte BOM-prefixed temporal seed is validated against strict bounds: not older than 30 days, not more than 15 minutes in the future, and not drifting more than 1 hour from block.timestamp. These checks ensure temporal entropy inputs are genuinely current and cannot be recycled from old mining sessions.

8.4 Signature Malleability Protection
MiningLib includes enhanced signature validation that rejects high-S ECDSA signatures, preventing the signature malleability attack vector that has historically affected some Ethereum contracts. Standard ECDSA signatures (65 bytes) are required. The high-S check uses the secp256k1 half-order threshold: s > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0 reverts.

8.5 Epoch Challenge Window
The 96-hour challenge window on BatchMiningModule epoch finalization provides a fraud-prevention layer on the batch path. Anyone can observe a committed Merkle root and dispute it before tokens are minted. This window uses L1 Ethereum block numbers (not L2), providing a timing guarantee that is independent of Arbitrum's sequencer.

8.6 Cap Enforcement
The hard cap is enforced at the token contract level, independent of any module. Even if a module is compromised or contains a bug that attempts to over-mint, the token contract's CapExceeded check provides a final guarantee that total supply can never exceed 2,000,000,000 TGBT. This check is in the token contract itself, which has no upgrade path.
 
9. Contract Architecture

The protocol consists of seven primary contracts and supporting libraries, all deployed on Arbitrum:

Contract	Role
TGBT.sol	ERC-20 token — hard cap, module authorization, stamp system, ossification
TemporalGradientCore.sol	Central registry — modules, output history, ossify(), pause
MiningModule.sol	Commit-reveal PoW — EIP-712 signatures, pool management, rate limiting
BatchMiningModule.sol	Epoch batch mining — Merkle roots, 96h challenge window, finalization
TokenomicsModule.sol	Emission schedule — epoch transitions, halving, bonus rewards, stale rewards
RateLimitModule.sol	Rate limiting — per-user token bucket and global sliding window
TokenomicsLib.sol	Stateless emission math — epoch, halving, reward preview (no storage)
MiningLib.sol	Cryptographic primitives — entropy hashing, signature validation, randomness

All modules inherit from ModuleBase, which enforces access through TemporalGradientCore's role system. No module can act outside its registered scope. The separation between the token contract's authorization layer and Core's role system provides defense in depth — a compromised module cannot escalate privileges at the token layer.
 
10. Conclusion

TGBT represents a synthesis of Bitcoin's monetary philosophy with Ethereum's programmability. Its hard cap and ossification model provide the strongest possible long-term supply guarantees available to an ERC-20 token. Its commit-reveal proof-of-work protocol and 96-hour challenge window ensure that every token in circulation represents genuine computational work anchored to verifiable external entropy.

The progressive decentralization model acknowledges the practical reality that complex systems require a bootstrap phase — while providing a credible, irreversible commitment to eliminating governance authority once the system reaches maturity. The ossify() topology check is not a promise. It is a constraint enforced in code.

Bitcoin closed its early miner window permanently. Industrial economics made individual CPU participation unviable. TGBT reopens that window — not by copying Bitcoin, but by applying the same foundational instinct to a different optimization target: proof of operational reality rather than proof of monetary settlement.

	Bitcoin will always need more power as it grows. Temporal Gradient becomes more valuable as it grows — without needing more power. Every new miner is a sensor, a witness, and a trust anchor. Every solution is a heartbeat. Every epoch is evidence.

— End of Whitepaper v1.0 —
