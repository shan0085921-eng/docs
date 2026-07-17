# Verisphere: A Truth-Staking Protocol
### White Paper — v15.0 (draft — under review)
**Date:** July 2026
**Contact:** info@verisphere.co

> **Scope.** This paper describes the Verisphere **protocol** only: the on-chain
> smart contracts in `core/`. It does not describe, and makes no representations
> about, any application, website, relay, market maker, pricing mechanism, or
> token-purchase service that any party (including Verisphere Ltd.) may build on
> top of the protocol. Those are separate services governed by their own terms.
> The protocol does not set, guarantee, or defend any market price for VSP.

---

## Abstract

Verisphere is a decentralized protocol for economically evaluating factual claims. Any participant may publish a claim on-chain and any participant may stake tokens to support or challenge it. Claims accumulate a Verity Score derived from the ratio and magnitude of stakes on each side. Evidence links between claims propagate truth-pressure through a directed graph, creating an interconnected epistemic structure where the credibility of each claim depends on the credibility of its evidence.

The protocol operates on the Avalanche C-Chain using the VSP ERC-20 token. All scoring, staking, and evidence-linking logic is implemented in upgradeable Solidity contracts. The protocol is permissionless: any front-end, API, or automated agent may interact with the on-chain contracts directly.

**The protocol does not adjudicate truth.** It has no oracle, no arbiter, and no privileged notion of what is true. Anyone may publish any assertion, however false, and anyone may stake for or against it. The protocol's only function is to make a position *cost something* when others are willing to stake against it: being wrong is expensive **only to the extent that other participants pay to make it so**. Verisphere does not decide what is true — it provides a transparent market in which participants attach their own economic conviction to claims, and it reports the running result.

---

## 1. Motivation

Existing information systems lack a mechanism for attaching economic cost to factual assertions. Search engines rank by engagement. Social platforms rank by virality. Encyclopedias rely on editorial consensus. Prediction markets handle binary, terminal events but cannot evaluate persistent, evolving claims such as "nuclear energy is environmentally sustainable" or "dietary saturated fat increases cardiovascular risk."

Verisphere addresses this gap by introducing a protocol where:

- Publishing a claim requires burning a fee, eliminating zero-cost spam.
- Supporting or challenging a claim requires staking tokens, imposing a cost on both truthful and untruthful assertions.
- Correct positions accrue value over time; incorrect positions lose it.
- Evidence relationships between claims are first-class on-chain objects with their own stake and credibility.

The protocol does not determine truth. It creates economic pressure that makes being wrong expensive and being right profitable, and it makes the resulting score transparent and auditable.

---

## 2. Protocol Overview

### 2.1 Posts

The atomic unit of the protocol is a **post**. Posts are of two types:

- **Claims**: standalone factual assertions (e.g., "Earth's mean radius is approximately 6,371 kilometers").
- **Links**: directed evidence relationships between two claims, annotated as either support or challenge.

Each post is identified by a sequential post ID. Post IDs begin at 1; zero is reserved as a null sentinel. Posts are immutable once created.

### 2.2 Claims

A claim is a single factual assertion stored as a UTF-8 string on-chain. Claims are deduplicated via case-insensitive, whitespace-normalized hashing (lowercase ASCII, collapse whitespace, trim, then keccak256). Attempting to create a duplicate claim reverts with `DuplicateClaim(existingPostId)`.

Publishing a claim requires transferring a posting fee of 1 VSP to the protocol, which burns it. The fee amount is governance-configurable.

### 2.3 Links

A link is a directed edge from one claim to another, annotated as either **support** or **challenge**. The direction is `from → to`, where "from" is the claim providing evidence and "to" is the claim receiving evidence.

Example: if claim S ("Earth is a spheroid") challenges claim F ("Earth is flat"), the link is `S → F` with `isChallenge = true`. This means S is the evidence provider and F is the evidence receiver.

Links are also posts and carry their own post ID, staking pool, and Verity Score. Duplicate links (same from, to, and challenge flag) are rejected. Self-loops are rejected.

The link graph permits cycles structurally. Two claims may challenge each other simultaneously. Cycles are not prevented at write time by the `LinkGraph` contract; instead, cycle handling occurs at score computation time in the `ScoreEngine` (see Section 4.3).

Creating a link also requires the posting fee, which is burned.

---

## 3. Staking

### 3.1 Positional Staking

Any participant may stake VSP tokens on any post (claim or link), but only on **one side** of any given post. A user with an existing support stake on a post cannot add a challenge stake to the same post (the contract reverts with `OppositeSideStaked`); they must withdraw fully and re-enter on the other side.

Each (post, side) pair holds at most **one consolidated lot per user**. A user's first stake on a side creates the lot at the back of the queue. Subsequent stakes by the same user on the same side merge into the existing lot: the lot's `amount` grows by the additional stake, and after every queue mutation the StakeEngine recomputes every lot's `weightedPosition` as the midpoint of its share of the side total — `weightedPosition = cumBefore + amount / 2`, where `cumBefore` is the running sum of amounts of lots earlier in the array. Earlier-entered users keep their array slot, so adding more stake to an existing lot enlarges the lot in place rather than moving the user to the back of the queue.

Stakes may be withdrawn at any time, in any amount up to the user's current lot balance. After a withdrawal, the StakeEngine recomputes every lot's `weightedPosition` from the new amounts, so a partial withdrawal slightly shifts the staker (and everyone after them in the array) toward the front of the queue. A user who withdraws fully retains their array slot with `amount = 0` (a "ghost lot"), which can be removed by a governance call to `compactLots`. A `lifo` parameter exists on the `withdraw` function for ABI compatibility but is ignored.

### 3.2 Staking Rate

Each lot accrues or loses value once per snapshot period (default: one day) according to a rate determined by:

1. **Truth pressure** — the verity magnitude (`|2A − T| / T`) determines the base strength of the economic force. When VS is exactly neutral (equal support and challenge), no economic effect occurs.
2. **Post size (participation)** — the post's total stake relative to a global reference (`sMax`) scales the rate. Larger posts face greater pressure. The factor is `participation = T / sMax`.
3. **Position weight** — each lot's weight is a continuous function of its weighted position in its side queue: `positionWeight = 1 − (lot.weightedPosition / sideTotal)`. Because positions are midpoints, a sole staker on a side has `weightedPosition = amount / 2` and therefore `positionWeight = 1/2`; the first of many earlier stakers (those with small `cumBefore`) earns close to the full rate, while later additions earn proportionally less.
4. **Governed bounds** — the annual rate is bounded between `rMin` and `rMax`, both governance-configurable. The current deployment uses 0% minimum and 100% maximum.

The base per-epoch rate (in RAY units, where RAY = 1e18) is:

```
rBase = rMin + (rMax − rMin) × verity × participation
```

Where `verity = |2A − T| × RAY / T` and both `rMin` and `rMax` have already been scaled from annual to per-epoch by the elapsed time (`× EPOCH_LENGTH × epochsElapsed / YEAR_LENGTH`).

Each lot's per-epoch change is then computed independently — there is no side-wide budget redistribution. For each lot:

```
delta = lot.amount × rBase × positionWeight / RAY
```

For a lot on the side aligned with the VS sign, `delta` accrues (VSP is minted to the StakeEngine and added to the lot). For a lot on the opposing side, `delta` is burned (VSP is destroyed and subtracted from the lot, capped so the lot never goes below zero). Because `positionWeight ∈ [0, 1]`, an individual lot's effective rate never exceeds `rBase`; a sole staker on a side earns `rBase / 2`.

**Position rescale.** After each snapshot's epoch math completes, the StakeEngine rescales all lots' `weightedPosition` values so that the maximum position is strictly less than `sideTotal`. This prevents the edge case where earlier stakers withdraw and shrink `sideTotal` below later stakers' positions, which would otherwise clamp those lots' `positionWeight` to zero indefinitely. The rescale preserves the relative ordering of all lots.

The global reference `sMax` is maintained by a top-3 post tracker: every stake or withdrawal updates this tracker and snaps `sMax` to the leader's total. As a result, during normal operation `sMax` always equals the largest active post's total stake — there is no slow decay to remove. A fallback exponential decay (configurable; currently 10% per epoch capped at 30 epochs) only runs in the corner case where the protocol has zero active posts, so a stale `sMax` cannot stay frozen forever after a complete unwind.

For the full normative specification, see `claim-spec-evm-abi.md`, Appendix A.

---

## 4. Verity Score

### 4.1 Base Verity Score

The base Verity Score reflects the direct stake ratio on a post.

Let `A` = total support stake, `D` = total challenge stake, `T = A + D`.

```
If A > D:   baseVS = +(A / T) × RAY
If D > A:   baseVS = −(D / T) × RAY
If A = D:   baseVS = 0
If T = 0:   baseVS = 0
```

Where `RAY = 10^18` (fixed-point scaling). The VS is clamped to `[−RAY, +RAY]`, corresponding to the range [−100%, +100%].

A post is considered **active** when its total stake meets or exceeds the activity threshold (governance-configurable, defaults to the posting fee). Inactive posts have no effect on other posts' effective VS.

### 4.2 Effective Verity Score

The effective Verity Score of a claim incorporates evidence from incoming links. Link contributions are **stake-weighted**: the parent claim's economic mass flows through the link to the child, not merely a percentage adjustment. This means the total stake on a parent claim determines how much influence it can exert through its evidence links.

#### 4.2.1 Credibility Gate

Only credible claims can influence other claims. A parent claim with effective VS ≤ 0 contributes nothing through its outgoing links. Its links become inert until the community rehabilitates the parent with direct support stakes.

Similarly, a link with base VS ≤ 0 (i.e., the community has challenged the evidence relationship itself) contributes nothing.

This rule prevents three classes of abuse:

- **Credibility laundering**: an attacker creates a deliberately false claim, lets it accumulate challenges, then uses it as a challenge link against a target. Without the credibility gate, the double-negative (discredited parent × challenge link) would produce a positive contribution, hijacking other users' challenge stakes.
- **Poisoned support**: an attacker links a toxic claim as support for a legitimate claim. When the toxic claim is challenged, the support link would become an attack via sign inversion. The credibility gate silences discredited parents regardless of link type.
- **Oscillation and cascades**: in cyclic graphs, sign inversions from negative-VS parents could create feedback loops and unpredictable downstream effects. The credibility gate prevents these by ensuring only positive-VS claims propagate influence.

#### 4.2.2 Stake-Weighted Contribution Formula

For each incoming link to claim C from parent P via link L:

**Step 1: Compute parent mass.**

The parent's economic mass represents its stake-weighted credibility:

```
parentMass = parentEffectiveVS × parentTotalStake / RAY
```

Where `parentEffectiveVS` is in the range `(0, RAY]` (always positive due to the credibility gate) and `parentTotalStake` is in token units (wei). The result is in token units and represents how much economic weight the parent carries.

**Step 2: Distribute across outgoing links.**

A parent's mass is distributed among its outgoing links in proportion to their stake, preventing duplication of influence:

```
linkShare = linkStake / sumOutgoingLinkStake
```

Where `sumOutgoingLinkStake` is the sum of total stake across all active outgoing links from P.

**Step 3: Apply link credibility.**

The link's own Verity Score reflects the community's assessment of whether this evidence relationship is valid:

```
contribution = parentMass × linkShare × linkVS / RAY
```

Where `linkVS` is the base VS of the link post. If `linkVS ≤ 0`, the link is discredited and contributes nothing.

**Step 4: Apply link direction.**

Challenge links invert the contribution:

```
if isChallenge: contribution = -contribution
```

A positive contribution adds to the child's support side. A negative contribution adds to the child's challenge side.

**Bounded fan-in.** For gas safety, the ScoreEngine processes at most `maxIncomingEdges` incoming links per claim and sums at most `maxOutgoingLinks` outgoing links per parent during the share computation. Both limits default to 64 and are governance-configurable. When a claim or parent exceeds its limit, the relevant edges are sorted by link stake descending — with ties broken deterministically by link postId ascending (older link wins) — and only the top-N participate. Lower-staked edges beyond the cap are inert: they neither contribute to the parent's denominator nor produce a numerator. This preserves conservation of influence (§4.4) under bounded fan-out: a parent's mass is fully and exclusively distributed across its top-N outgoing links. Off-chain indexers that recompute scores should apply the same sort-and-cap rule with the same tiebreak to match on-chain behavior.

#### 4.2.3 Effective VS Computation

After accumulating all incoming link contributions:

```
totalSupport = directSupport + sum(positive contributions)
totalChallenge = directChallenge + abs(sum(negative contributions))
pool = totalSupport + totalChallenge

effectiveVS = (totalSupport - totalChallenge) / pool × RAY
```

The result is clamped to `[−RAY, +RAY]`.

#### 4.2.4 Example

Claim A has 2 VSP support (VS = +100%). Claim B has 1 VSP support (VS = +100%). A challenges B via a link with 2 VSP support (link VS = +100%).

- parentMass(A) = 1.0 × 2.0 / 1.0 = 2.0 VSP
- sumOutgoing(A) = 2.0 (one outgoing link)
- linkShare = 2.0 / 2.0 = 1.0
- contribution = 2.0 × 1.0 × 1.0 = 2.0
- isChallenge → contribution = -2.0
- B: totalSupport = 1.0, totalChallenge = 0 + 2.0 = 2.0
- pool = 3.0
- effectiveVS(B) = (1.0 - 2.0) / 3.0 = **-33.3%**

The claim with 1 VSP support is pushed negative by the 2 VSP challenger. To defend B, participants can: add direct support to B, challenge claim A (reducing its VS and therefore its mass), or challenge the link itself (reducing its VS to silence it).

### 4.3 Cycle Handling

The link graph permits cycles structurally; the `LinkGraph` contract does not enforce acyclicity at write time. Acyclicity of the *score computation* is instead enforced at read time by the `ScoreEngine`:

1. When computing `effectiveVS(C)`, the engine maintains a stack of post IDs currently being computed (passed by reference through recursive calls).
2. Before recursing into a parent claim, the engine scans the stack. If the parent's post ID is already present, the recursion would close a cycle; the engine returns `0` for that parent's contribution rather than recursing.
3. A hard depth limit of 32 provides additional safety. Beyond this depth, contributions are truncated to zero regardless of cycle membership.

When a cycle is detected, only the cycled post's contribution is zeroed for that path. Other incoming edges of the same parent still compute normally. For example, if computing VS(F) encounters chain F→A→S→F, then F's contribution back to S along that path is 0, but A's other incoming edges (if any) are unaffected.

Combined with the credibility gate (Section 4.2.1), cycles are further stabilized: if a claim's effective VS drops to zero or below during computation, it ceases to influence its neighbors, preventing oscillatory feedback. The result is that the effective VS function is well-defined and bounded on any directed graph the protocol can produce, not merely on a DAG.

### 4.4 Conservation of Influence

A claim's economic mass is finite and is distributed — not duplicated — across its outgoing links. If a parent has mass M and three outgoing links with equal stake, each receives M/3. Adding more outgoing links from the same parent dilutes each link's share.

Under bounded fan-out (§4.2.2), this rule is enforced strictly: only the parent's top-`maxOutgoingLinks` outgoing links by stake participate in the distribution. Links beyond the cap are inert — they contribute zero to the target's effective VS and do not appear in the parent's denominator. Two consequences follow:

- The sum of `linkShare` across all of a parent's outgoing links is always ≤ 1.0, even when the parent has more outgoing links than the cap.
- Link spam past the cap is fully self-defeating: a low-stake "spam" link that fails to displace one of the top-N has zero influence. The attacker has burned their posting fee and locked stake for no effect.

This ensures:
- A single claim cannot amplify influence beyond its own stake-weighted credibility.
- The cost of meaningful influence scales with the stake required to maintain both the parent claim and the link, *and* to keep the link competitive against other outgoing links.
- Link spam is self-defeating: each additional link either dilutes the attacker's influence per target (within the cap) or contributes nothing (beyond the cap).

---

## 5. Token Economics

### 5.1 VSP Token

VSP is the native ERC-20 token of the protocol, deployed on Avalanche C-Chain. It has governance-controlled mint and burn functions managed through an Authority contract. VSP supports ERC-2612 permit, enabling gasless approvals.

### 5.2 Posting Fee

The posting fee is 1 VSP, burned upon post creation. The fee amount is governance-configurable.

The posting fee serves two purposes:
- Spam prevention: imposes a cost on publishing claims.
- Activity threshold: a post must accumulate total stake ≥ the activity threshold (which defaults to the posting fee) to become active and influence other posts.

### 5.3 Economic Properties

- **Deflationary pressure**: posting fees are burned, reducing supply.
- **Inflationary pressure**: correct stakes accrue value (minted by the StakeEngine).
- **Equilibrium**: the balance between creation (burning) and staking (minting) is governed by protocol parameters.

The mint and burn of VSP that back stake accrual and decay are performed under the
`Authority`'s minter/burner roles. In operation, an automated off-chain process
holds these roles and executes the periodic snapshot that mints to accruing lots and
burns from decaying ones, according to the on-chain rules described in §3.2. The
authority to mint and burn is governed as described in §6.2; the roles are
revocable by governance.

---

## 6. Smart Contract Architecture

All contracts are deployed as UUPS upgradeable proxies behind a governance-controlled Authority.

| Contract | Purpose |
|----------|---------|
| VSPToken | ERC-20 token with ERC-2612 permit, Authority-controlled mint and burn |
| PostRegistry | Creates claims and links, burns posting fees, stores post metadata |
| LinkGraph | Stores directed evidence edges, enforces self-loop and duplicate prevention (cycles permitted) |
| StakeEngine | Manages per-post consolidated lots, computes positional rates, handles withdrawals |
| ScoreEngine | Computes base and effective Verity Scores with stake-weighted propagation, cycle-safe |
| ProtocolViews | Read-only aggregation of claim summaries, edge data, and scores |
| PostingFeePolicy | Governance-configurable posting fee |
| StakeRatePolicy | Governance-configurable staking rate bounds |
| ClaimActivityPolicy | Defines the minimum stake threshold for post activation |
| Authority | Role-based access control (minter, burner, governance roles) |

### 6.1 Meta-Transaction Support

All governed contracts (PostRegistry, StakeEngine, LinkGraph) inherit `ERC2771ContextUpgradeable`, which allows a trusted forwarder to submit transactions on behalf of users. The trusted forwarder address is set at deployment and can be updated via UUPS proxy upgrades.

VSPToken supports ERC-2612 permit, enabling signature-based approvals without a separate on-chain transaction.

These primitives allow third-party services to offer gasless interaction with the protocol. The protocol itself does not operate a relay or forwarder — it provides the on-chain hooks that make them possible. Users may always interact with the contracts directly using their own wallet and gas.

### 6.2 Governance

On the production (mainnet) deployment, governance authority over the protocol is
held by a `TimelockController` whose sole proposer and executor is a multi-signature
(2-of-3) operations Safe. The timelock enforces a minimum delay (no less than two
days on mainnet) between the scheduling and the execution of any privileged action,
and the timelock admin role is renounced at deployment, so there is no deployer
backdoor and no unilateral fast path.

**Governance handoff is part of deployment, not an optional later step.** The
deployment procedure deploys the timelock, then transfers ownership of every
governed contract — including the `Authority` — from the deploying account to the
timelock via a two-step (propose/accept) transfer executed through the Safe. A
mainnet deployment is **not considered complete** until this handoff has occurred
and on-chain ownership of `Authority` resolves to the timelock. Until that point the
contracts remain deployer-owned and the deployment is provisional.

(Non-production test deployments may retain deployer ownership for iteration; that
is a property of a test environment, not of the protocol as deployed for real use.)

Through this timelocked, Safe-controlled governance, the following may be modified:
- Posting fee amount
- Staking rate bounds (min and max APR)
- Activity threshold
- Snapshot period and ScoreEngine fan-in limits
- Contract implementations (via UUPS proxy upgrades)
- Authority roles

---

## 7. Design Properties

### 7.1 Permissionless

Any address may create claims, create links, and stake. No registration, reputation, or identity is required. Front-ends, bots, and automated agents interact with the same contracts as human users.

### 7.2 Composable

The protocol exposes all state through standard Solidity view functions. Third-party applications may build on top of the protocol: alternative front-ends, analytics dashboards, AI-powered truth-checking tools, and cross-chain bridges.

### 7.3 Non-Finalizing

Claims never "resolve." The Verity Score is a continuous, live signal that reflects the current state of economic commitment. New evidence, new stakes, and new challenges can shift any claim's score at any time. This makes the protocol suitable for persistent, evolving knowledge — not just terminal predictions.

### 7.4 Adversarial

The protocol is designed for adversarial participants. There is no assumption of good faith. Economic incentives align with truthful behavior: being right is profitable; being wrong is costly. The credibility gate (Section 4.2.1) ensures that discredited claims cannot be weaponized through the evidence graph. The protocol does not enforce truth — it creates conditions under which truth is economically favored.

---

## 8. Deployment

The protocol is deployed on Avalanche C-Chain (chain ID 43113 for testnet, 43114 for mainnet). Avalanche provides sub-second finality, EVM compatibility, and low transaction costs suitable for interactive staking.

The protocol may optionally be deployed on a dedicated Avalanche Subnet for isolated throughput and custom gas economics.

---

## 9. Conclusion

Verisphere defines a minimal, permissionless protocol for attaching economic consequence to factual assertions. Claims compete in an open market of support and challenge. Evidence links create a directed graph where credibility propagates through stake-weighted connections. Only credible claims — those with positive community support — can influence others, preventing abuse through double-negative exploits or poisoned associations. The Verity Score provides a transparent, continuously updated signal of economic consensus.

The protocol does not determine truth. It makes truth economically consequential.
