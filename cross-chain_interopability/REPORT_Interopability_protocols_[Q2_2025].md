Detailed comparative report on interoperability protocols including Wormhole, Axelar, Across, Chainlink CCIP, LayerZero, Synapse, and Skate—along with other major solutions used for cross-chain communication and token transfers.

This report includes:

* A foundational overview of each protocol
* Technical and architectural design (oracle-based, light clients, intent-based, etc.)
* Cross-chain token and data transfer mechanisms
* Unified liquidity support and general message-passing capabilities
* Transfer durations and fee structures
* Security models and trust assumptions
* Summary comparisons, advantages/disadvantages, and tables for side-by-side analysis



# **Cross-Chain Interoperability Protocols: Comparative Report**

## **Wormhole**

Wormhole (launched 2021 by Jump Crypto) is a leading general-purpose cross-chain messaging protocol. It began as a Solana–Ethereum token bridge and evolved into Wormhole V2, an arbitrary message layer spanning 20+ chains (EVM and non-EVM). Its core design uses a **guardian network**: currently 19 independent node operators (“Guardians”) that monitor source-chain contracts and collectively sign off on cross-chain events. Guardians use a t-Schnorr threshold signature scheme (2/3 threshold) to produce “Verifiable Action Approvals (VAAs)”. Each chain has a Wormhole *core contract* (a Router) and a Token Bridge contract: when a user deposits or burns tokens on chain A, Guardians observe this and emit a signed VAA to the Token Bridge on chain B. The bridge then releases or mints the corresponding wrapped asset on chain B. Wormhole also supports **Token Transfers with Messages**, enabling arbitrary data payloads to accompany transfers.

Wormhole’s model is **lock-and-mint** (or burn-and-release) with fragmented liquidity: assets are escrowed on the source chain and wrapped on the destination. Thus there is no unified liquidity pool – each chain’s Wormhole vault holds its own tokens. Transfers historically required waiting for multiple confirmations (often a few minutes) plus Guardian signature aggregation. In practice bridging could take on the order of 5–15 minutes under the original Wormhole, though fees were generally just gas plus a small bridge fee. (Wormhole’s new **Settlement** product (2025) offers *near-instant* bridging by using an intent/solver model, but this is still in pilot.)

In addition to its classic lock‑and‑mint “Token Bridge,” Wormhole now offers:

**Native Token Transfers (NTT)** – NTT lets issuers deploy a single native token contract across multiple chains. Transfers are driven by VAAs (no wrapped assets), preserving on‑chain fungibility and simplifying custody.

**Wormhole Connect** – a plug‑and‑play React widget for end‑users to initiate cross‑chain transfers without custom UI work.

**Wormhole Settlement** – an intent‑based, fast‑finality layer for institutional flows, using solver networks atop the Guardian messaging layer.

**MultiGov** – a hub‑and‑spoke governance framework for DAOs to coordinate proposals and upgrades across chains

Native Token Transfers (NTT) Architecture  
NTT is built around Managers and Transceivers on each chain.  
Managers (one per token) handle lock/burn logic, emit TransferSent events, enforce customizable rate limits (queuing or reverting excess transfers), and verify message authenticity by collecting a threshold of signatures from Transceivers.

Transceivers route messages off‑chain—using Wormhole relayers or custom back‑ends—quoting delivery fees, forwarding VAAs, and emitting SendTransceiverMessage events.

On arrival, Managers on the destination chain verify the VAA attestation (against the M‑of‑N threshold), then mint or unlock the native tokens in a single step.  
This design requires no wrapped assets and keeps supply synchronized across deployments. Custom transceivers can plug in additional attestation layers if desired

| Aspect | Token Bridge (Wrapped) | Native Token Transfers (NTT) |
| :---- | :---- | :---- |
| **Asset Model** | Wrapped tokens pegged 1:1, held in per‑chain vaults | Single native token contract deployed on each chain |
| **Custody** | Centralized vaults hold collateral | No vaults—mint/burn logic governed by VAA attestations |
| **Metadata Handling** | Requires up‑front *attestation* (AssetMeta payload) | Metadata intrinsic to native contract, no separate attestation step |
| **Decimals & Precision** | Truncates to 8 decimals to ensure cross‑chain compatibility | Preserves full precision as defined in native contract |
| **Liquidity Fragmentation** | Each chain’s vault holds its own liquidity | Liquidity is unified at the token‑contract level, issuer‑controlled |
| **Security Model** | Guardian multisig (PoA, ≥2/3) | Same Guardian consensus for VAA, plus GlobalAccountant supply checks |
| **Use Cases** | User bridges wrapped assets (DEX liquidity, yield) | Issuers publish truly native multichain tokens (e.g. WBTC) |

In summary, Wormhole’s Token Bridge remains the go‑to for wrapped‑asset transfers where custodial vaults are acceptable and broad chain support is needed. By contrast, NTT removes vault custodianship entirely—issuing a single canonical token logic across all chains, with cross‑chain transfers driven by the same VAA mechanism but without extra wrapping steps.

Wormhole’s **security/trust** model is hybrid: it is a permissioned Proof-of-Authority system with no tokenized stakes. The trust assumption is that at least 2/3 of the Guardians are honest and keep their private keys secure. If a majority of guardians collude or are compromised, they could mint arbitrary assets. A real-world exploit illustrates this risk: in Feb 2022 a smart-contract vulnerability (in the Solana bridge contracts) allowed an attacker to bypass signature checks and mint 120,000 wETH (\~$320M) on Solana without collateral. This hack was due to a coding bug (not a guardian collusion), but it highlights that Wormhole’s security relies on both guardian integrity and correct implementation. Wormhole aims to enhance security over time via threshold signatures and (in future) ZK-based verification. In summary, Wormhole offers wide EVM & non-EVM support, arbitrary messaging, and well-tested bridging, but relies on a semi-trusted guardian set and careful audits of its contracts.

## **Axelar**

Axelar is a Cosmos-SDK blockchain launched in 2022 to act as a **universal cross-chain hub**. It provides general message passing (GMP) and native token bridging across 50+ chains (EVMs, Cosmos-Tendermint chains, Bitcoin, etc.). In Axelar’s model, there is a *Axelar Mainnet* (a Tendermint PoS chain) plus **Gateway contracts** on each connected chain. Axelar validators (up to dozens, currently capped at 50\) run PoS consensus on the Axelar chain and a “Cross-Chain Gateway Protocol” (multi-party signing) for external events.

**Transfer workflow:** To send tokens/data from chain A to B, a user or dApp calls the Axelar *Gateway* contract on chain A with the payload (which can include native-token lock, mint or arbitrary calldata). Axelar validators watch the source gateway, come to 2/3 consensus that the event occurred, then threshold-sign a packet of commands (using multi-party cryptography). A relayer (permissionless microservice) then submits this signed command batch to the destination gateway on chain B, which executes it (unlocking/minting tokens or invoking the target contract). In effect, Axelar uses a **hub-and-spoke** model: assets are locked on A and an equal amount of canonical tokens are minted on B. After use, tokens can be burned on B to release the original on A. Axelar also launched an *Interchain Token Service* (ITS) to allow truly “native-like” tokens across chains, but the underlying mechanism is similar.

Axelar supports arbitrary cross-chain messages (“send any payload”) via its **General Message Passing (GMP)** framework. Developers interact through Axelar SDKs/APIs and can even deploy cross-chain smart contracts. It offers features like automatic gas payment (conversion to AXL, the network’s token) and single-deposit addresses. **Fees and speed:** Axelar transactions require paying gas on source (and optionally AXL fees), and the relay may incur gas on destination. Users typically only pay the source-chain fees, as Axelar’s Gas Service may subsidize or batched actions. Consensus finality on Axelar (Tendermint) is \~1s, but practical bridge latency includes waiting for confirmations on each chain and for validator consensus (on the order of 1–3 minutes typical). A notable design is *on-chain gas quoting*: Axelar validators create a single signature authorizing the transfer, keeping on-chain calls small.

**Security/trust:** Axelar’s cross-chain security inherits its PoS model: trust that ≥2/3 of Axelar validators act honestly. Validators stake AXL tokens, have incentives via staking rewards, and governance can add more validators. Axelar also embeds risk controls: it leverages Cosmos IBC security for compatible chains, can freeze transfers on a chain under attack, and uses rate limits on large transfers. In short, Axelar is more trust-minimized than a federated bridge: it has its own blockchain consensus, on-chain governance, and token economics, but still assumes majority honesty of a validator set. Its use of separate gateway modules per chain isolates chains from each other, reducing cross-impact.

## **Across Protocol**

Across is a novel “intent-centric” bridge (2023 launch) designed for *speed and UX*. Rather than simple lock/mint, Across asks users to specify an **intent** (a high-level goal like “swap A→B”) and then uses a network of *relayers* to fulfill it. The user deposits tokens into a **Spoke Pool** on the source chain, along with their instructions and a fee offer. Off-chain relayers watch these Spoke Pool deposits, and the first to bid and fulfill the intent (by sending funds to the user on the destination chain) wins. The user receives tokens on chain B almost immediately (minus fees) and no longer needs to wait. Later, the relayer submits a proof of the completed transfer to an **Optimistic Oracle (OO)**, which verifies the deposit and relayer’s work. Once the proof is accepted, the relayer is reimbursed from a centralized **Hub Pool** on Ethereum (funded by liquidity providers).

Thus Across splits bridging into phases: **(1) User deposit to Spoke Pool; (2) Relayer fronts the swap to user; (3) Optimistic settlement and reimbursement from the Hub Pool.** This decoupling (via optimistic settlement) allows very fast “\<1 minute” completion. Across effectively offers **immediate bridging**: the user sees funds on B as soon as the relayer sends them. Over time (typically minutes or hours), the system settles on-chain via the optimistic oracle, giving challengers a chance to dispute fraud.

Across supports arbitrary payloads in intents (e.g. “bridge and swap on DEX”), but its primary focus is token/asset transfers. Liquidity is semi-unified through the Hub Pool: LPs stake assets on Ethereum which back relayers, but the actual transfer happens via relayer networks rather than a single global pool. **Fees and speed:** Across advertises lowest fees and fastest times: \~\<$1 to bridge 1 ETH, with average fill under 1 minute. Users bid their own fee to relayers, so actual cost depends on market competition and gas.

**Security/trust:** Across uses optimistic security: it assumes relayers act honestly because fraud would be challenged. The Optimistic Oracle (likely UMA’s OO) is the arbiter. Relayers must eventually prove their action on-chain or be penalized. If a relayer cheats (e.g. fakes a transfer), other participants can dispute before the timeout. Thus trust is placed in the threat of challenge rather than a fixed validator set. This model risks stealing only if no one watches the oracle period. All funds are held in audited smart contracts (Spoke Pools, Hub Pool) and the OO’s logic. Potential risks include oracle delays or collusion among a majority of relayers (unlikely if open). Overall Across trades some complexity and centralization (a single Hub Pool) for near-instant settlement.

## **Chainlink CCIP**

Chainlink’s **Cross-Chain Interoperability Protocol (CCIP)** is a decentralized messaging and token-transfer framework launched 2023\. CCIP leverages the existing Chainlink decentralized oracle networks (DONs) to secure cross-chain commands. Conceptually, a user calls ccipSend on a **Router** contract on chain A with the desired payload (token transfer or message). The Chainlink *commitment layer* (a DON) observes this and collectively signs a packet of actions for chain B. Another *execution layer* DON on chain B then submits the signed packet to the Router on B for execution. An independent **Risk Management Network (ARM)** monitors for anomalies during this process. In practice, each cross-chain “lane” uses three separate Chainlink oracle networks: one to commit the message, one to execute it, and one (ARM) to supervise security.

For token bridging, CCIP typically uses a **burn-and-mint** mechanism. The user approves the Router to handle their tokens, then calls a CCIP transfer. The source Router locks/burns the tokens and emits an event. After oracle signatures, the destination Router **mints** the same amount of the token (or its “Cross-Chain Token” version) to the recipient. CCIP also allows arbitrary data to be sent cross-chain (via data fields).

**Coverage:** CCIP is designed to be chain-agnostic and is live on many networks (Ethereum, BNB, Polygon, Arbitrum, Solana, etc.). It supports any combination of chains via the Router abstraction. **Fees and speed:** Transactions involve Chainlink oracle fees plus gas on both chains. There is a service fee (roughly a few hundredths of a percent) plus base gas costs. Execution time includes waiting for oracle consensus (tens of seconds to minutes) and on-chain finality. The exact latency depends on the chain and required confirmations, but CCIP aims for near real-time.

**Security/trust:** CCIP’s design is **highly decentralized**. Chainlink emphasizes “Level-5 security”: each message is verified by multiple independent DONs, and an external Risk Network applies configurable policies. The trust assumption is that at least a threshold of oracle nodes in each network are honest. Because there’s no single key, a rogue oracle or even a colluding group still faces checks by the other layers. If a chain reorg or attack is detected, the Risk Network can pause or reject transactions. In contrast to bridges with fixed guardians, CCIP’s multi-layer oracle model and proactive monitoring make it one of the most secure architectures. Its main risks are a chainlink node compromise (highly improbable) or smart-contract bugs.

## **LayerZero**

LayerZero is an omnichain interoperability protocol that has significantly evolved to its current **v2 architecture**, designed to facilitate secure, efficient, and customizable communication across disparate blockchain networks. Its primary objective is to enable smart contracts to seamlessly interact—reading from and writing state to different blockchains—fostering a unified environment for decentralized applications (dApps).1

At its core, LayerZero v2 supports **Generic Message Passing (GMP)**, a foundational primitive that empowers applications to transmit and receive arbitrary data across a fully-connected mesh of blockchains.1 This capability extends beyond simple asset transfers, allowing for complex state transitions and interactions that transcend the boundaries of any single chain.1

### **Core Architectural Innovations**

The most significant architectural advancement in LayerZero v2 is the **Separation of Verification and Execution**. Unlike v1, where the Relayer performed both verification and execution, v2 distinctly separates these functions. Verification is now handled by a customizable **Security Stack**, composed of **Decentralized Verifier Networks (DVNs)**, while Execution is managed by independent **Executors**. This decoupling fundamentally enhances scalability, efficiency, and resilience by ensuring that network liveness issues (e.g., an Executor being temporarily unresponsive) do not compromise the core security and validation process of messages.3

LayerZero v2 maintains the core philosophy of an **Immutable Protocol / Configurable Edge**. The underlying protocol contracts are non-upgradeable, audited, and battle-tested, having secured over $50 billion in transfer volume. This immutability provides a stable and secure foundation. Application-specific smart contracts built on LayerZero remain highly configurable atop this immutable layer, allowing developers to ship new features without directly modifying the core transport protocol.1

### **Modular Security Framework**

V2 introduces a sophisticated **modular security model** that differentiates between Intrinsic and Extrinsic Security. **Intrinsic Security** refers to the security inherent in the core LayerZero protocol itself, which remains immutable and non-upgradeable once deployed. It guarantees fundamental properties like lossless, once-and-final delivery of data packets.3 **Extrinsic Security**, conversely, is adaptable and designed to evolve with new capabilities and verification algorithms. It encompasses external infrastructure components such as the Decentralized Verifier Networks (DVNs) and Executors, providing a flexible layer of security that can be customized and upgraded.3

A cornerstone of v2 is the ability for applications to configure their own **Customizable Security Stack with Decentralized Verifier Networks (DVNs)**. Developers can choose from a diverse set of DVNs, which are independent entities responsible for validating messages between chains for security and integrity.1 V2 introduces an advanced verification model known as the "**X of Y of N**" approach. This allows applications to arbitrarily combine DVNs for message validation, moving beyond the perceived 2/2 multisig comparison of v1's oracle/relayer setup. This mechanism significantly enhances the trust model and resilience against downtime.3 This customizable approach empowers developers to "**dial-in security per message**" 2, allowing them to balance security assurances, transaction speed, and associated costs based on their specific application needs.4

Both DVNs and Executors operate as **Permissionless Infrastructure** within the LayerZero ecosystem. This open design encourages competition among service providers, fostering competitive pricing and driving improvements in the quality of service for message verification and execution. Any entity can run these components.3

### **Advanced Messaging and Execution**

LayerZero v2 introduces greater flexibility in message processing through **Ordered vs. Unordered Message Delivery**. Applications can now choose between ordered message delivery, which ensures sequential execution, and unordered delivery, which enables simultaneous execution of messages, depending on the application's specific requirements for concurrency and state consistency.4

A powerful new feature in v2 is the support for **Composed Messages**. This enables developers to send a single cross-chain message that can trigger multiple distinct actions on different contracts on the destination chain. The Executor processes these steps either sequentially or concurrently as defined by the application.2

In v2, the responsibility for transaction execution is shifted to the **End-User Execution**. This reinforces the permissionless nature of transactions and facilitates advanced features like gas abstraction, where users can pay for cross-chain gas in a single native token. This self-executing model ensures that operations can be registered and completed without reliance on intermediaries, enhancing decentralization.3 **Executors** are off-chain services that perform the final step of executing messages on the destination chain, but only *after* the message has been thoroughly verified by the chosen Security Stack (DVNs).2

### **Developer Experience and Unified Semantics**

LayerZero v2 expands its architectural primitives to offer more robust cross-chain capabilities. These **New Primitives** include: **Omnichain Message Passing (GMP)** 1; **Omnichain Tokens (OFT & ONFT)**, unified token standards that enable seamless cross-chain transfer of fungible and non-fungible tokens 1; **Omnichain State Queries (lzRead)**, which allows smart contracts to securely request and retrieve on-chain state data directly from other blockchains 1; and **Omnichain Composability**, enabling complex, multi-step workflows across chains by decoupling security from execution.1

A critical enhancement in v2 is its **Expanded Chain Support**, including non-EVM chains. LayerZero now offers compatibility with over 120+ different EVM, Solana, Move, and TON compatible blockchains, significantly broadening its interoperability scope.1

**Backward Compatibility and Migration** are key considerations. Endpoint V2 is designed to be fully backward compatible with the original Endpoint V1. Existing applications deployed on V1 can access the new Security Stack and independent execution features by configuring their application's Message Library to UltraLightNode301, making migration optional but beneficial.5

To improve clarity and align with the expanded support for non-EVM chains, several key terms have undergone **Terminology Updates**.5 These revisions are detailed in the table below:

**Table 1: LayerZero Terminology Updates (v1 vs. v2)**

| LayerZero v1 Term | LayerZero v2 Term | Description |
| :---- | :---- | :---- |
| chainId | eid (Endpoint ID) | A unique identifier for each LayerZero Endpoint contract, now generalized to accommodate both EVM and non-EVM chains for routing messages. 5 |
| adapterParams | \_options (Message Options) | A byte array of arguments interpreted by the MessageLib, allowing developers to provide specific instructions for message execution, such as gas payment details or handling for specific messaging types. 5 |
| User Application (UA) | Omnichain Application (OApp) | The new term for applications built on LayerZero, emphasizing their inherent ability to operate across multiple chains. 5 |
| srcAddress | sender | A more generic term for the message originator, accommodating various chain formats (e.g., an address on EVM, a public key on non-EVM chains). 5 |
| dstAddress | receiver | A more inclusive term for the message recipient, catering to both EVM and non-EVM chain formats. 5 |
| payload | message | Refers specifically to the content of the packet, providing a clearer distinction from the Global Unique Identifier (GUID) within a message. 5 |

LayerZero also offers a suite of **Developer Tooling** to streamline development, including LayerZero Scan (a comprehensive block explorer), TestHelper (for simulating cross-chain transactions in Foundry tests), and Hardhat Toolbox (for building, configuring, and deploying OApps locally).2

### **Liquidity, Speed, and Security**

**Liquidity:** LayerZero itself does not provide liquidity. Bridges and applications built on LayerZero (e.g., Stargate) handle tokens separately, as LayerZero's role is strictly generic messaging.2

**Speed and Fees:** LayerZero messages typically finalize within a couple of minutes, primarily dominated by waiting for block confirmations and the processing by the chosen DVNs and Executors.4 The user generally only pays gas on the source chain call; LayerZero does not require separate fee tokens.4

**Security:** LayerZero v2's security hinges on its modular security framework. The **trust model** is now highly customizable, relying on the configured "X of Y of N" DVN combination to verify messages. The separation of verification and execution means that even if an Executor experiences downtime, the message verification process remains secure and unaffected.3 The immutable core protocol provides intrinsic security against fundamental issues.1 This design empowers applications to control their trust assumptions, balancing security assurances with performance and cost.2

## **Synapse Protocol (additional)**

Synapse is a widely used cross-chain bridge (especially for stablecoins) that uses on-chain liquidity pools. Each supported token has a *Synapse Router* contract and “bridge tokens” on each chain. Users swap from the canonical token to a chain-specific token (like sUSD on each chain) using an AMM. To bridge, a user sends token on chain A to the Synapse Router, which credits them an equivalent on chain B drawn from the liquidity pool. For example, bridging ETH L1→L2, a user gets hETH on L2 minted from the pool, then swaps it via AMM to real ETH. The pool on each chain is funded by liquidity providers, so bridging is instant (no waiting on cross-chain consensus). Fees are moderate (AMM swap fees \+ small protocol fee).

**Security:** Synapse relies entirely on its on-chain contracts and pools. There are no external signers or oracles. The protocol has suffered a few incidents (e.g. a front-end hack in Aug 2022), but its core contracts have been audited. As with any AMM, key risks are contract bugs and impermanent loss for LPs.

## **Connext**

Connext is a trust-minimized liquidity bridge for EVM chains. Its current NXTP protocol (launched 2021\) uses a **decentralized router network** rather than external validators. Liquidity providers (“Routers”) deposit collateral on each chain. When a user sends tokens A→B, they first broadcast an intent and receive bids from routers. The user picks a router’s bid and submits their transfer to the *TransactionManager* on chain A, locking their tokens with that router. The router simultaneously locks corresponding tokens on chain B (minus the fee it will earn) in B’s TransactionManager. Once both sides are prepared, the user (or a relayer) sends a signed “fulfillment” message through Connext, which prompts the router to release the tokens to the user on chain B and unlock the router’s original collateral on A. This mechanism is fully **non-custodial**: user funds are locked in smart contracts, and router collateral is locked as well. Connext guarantees that funds cannot be stolen even if routers fail: if the protocol’s dispute mechanism triggers, users can recover directly from the contract.

Connext supports bridging arbitrary calldata (cross-contract calls) as well as tokens. The ecosystem is any-to-any EVM (including L2s). **Speed:** Transfers execute in a few seconds once routers respond. Fees are competitive (router bids typically reflect \<0.1% plus gas).

**Security:** Connext’s trust model is that *no external validators exist*. It relies only on on-chain locks and economic incentives. A router cannot cheat because its funds are locked and it risks losing collateral. In effect, security equals Ethereum’s security for the underlying token transfers. Risks include the rare possibility that a router colludes to drop a message, in which case timeout/cancellation mechanisms allow refunds. Connext is audited and battle-tested; it has not suffered any major losses in its NXTP phase.

## **Celer Network (cBridge)**

Celer’s **cBridge** is a large multi-chain bridge (40+ chains, $14B volume as of 2024). Its core is the **State Guardian Network (SGN)**: a Cosmos-based PoS chain of validators (initially \~100) that collectively verify cross-chain transfers. When a user bridges token A→B, they call send() on cBridge’s contract on chain A. Their deposit is recognized by SGN (via its on-chain relayers). The SGN validators then sign a message authorizing minting or release on chain B, which is submitted to cBridge’s contract on B.

cBridge actually implements two models: **xAsset** (lock-and-mint) and **xLiquidity**. xAsset works like Wormhole: source funds are locked (or burned) and minted on dest via SGN signatures. xLiquidity uses on-chain pools (similar to Hop): it maintains liquidity pools of tokens on each chain, so users can swap via those pools for near-instant transfers. For example, bridging stablecoin, a user can use the liquidity pool on A to receive instantly from the pool on B; SGN finalizes the chain communication behind the scenes.

**Fees & Speed:** cBridge offers fast transfers. If liquidity is available (xLiquidity), the user receives funds almost instantly (pool swap) and SGN finality happens seconds later. If not, xAsset path takes longer (waiting for enough SGN confirmations). Fees are low: a base fee \+ a small protocol fee (often \<\<1%).

**Security:** SGN’s PoS chain secures cBridge. In practice, trust that \>2/3 of SGN validators act honestly. Jump Crypto’s analysis found a vulnerability in SGN’s election code (fixed without loss), highlighting that a single malicious validator *could* compromise the bridge. Unlike pure AMM bridges, cBridge has this centralized element. However, SGN is publicly governed and has bug bounties. The on-chain contracts are audited. Other risks include smart contract bugs. Notably, a 2022 front-end attack on cBridge (phishing users’ approvals) cost some funds, but cBridge’s core remained secure.

## **Hop Protocol**

Hop is a specialized rollup-to-rollup token bridge (launched 2021\) using **AMM-like liquidity pools**. For each canonical token (ETH, DAI, USDC, etc.), Hop issues “hTokens” on each supported chain (e.g. hETH on Optimism, hETH on Arbitrum, etc.). Users do not handle hTokens directly; they always transact in the normal tokens. Under the hood, when bridging, Hop mints an hToken on the destination chain and then immediately swaps it via a built-in AMM pool to the real token. For example, bridging ETH L1→L2: the user deposits ETH on L1, Hop records it; the user then withdraws the equivalent hETH on L2; Hop’s AMM pool swaps hETH for real ETH on L2, delivering it to the user.

Thus Hop’s model is **pool-based with intermediary tokens**. Liquidity providers stake both hTokens and real tokens in the pools on each chain. Transfers are effectively instant (just AMM swap time) and “trust-minimized” (no off-chain validator key needed).

**Speed & Fees:** Because Hop uses on-chain pools, users receive funds as soon as transactions confirm (no extra delay). Typical transfer times are a few seconds plus block finality. Fees are two parts: (1) swapping through the AMM pool (protocol fee \~0.1% plus slippage) and (2) a small “destination gas” fee charged in token. These are generally low (\~0.1–0.3% total).

**Security:** Hop’s contracts are audited and its model is non-custodial. The main risk is impermanent loss to LPs or edge cases in the AMM. No known hacks of Hop’s core mechanism have occurred. The trust assumption is minimal: only the Ethereum security and smart-contract correctness.

## **Other Protocols**

* **IBC (Inter-Blockchain Communication):** The Cosmos ecosystem’s native interoperability protocol. IBC is a protocol spec (not a single project) that uses **light clients** on each chain to verify cross-chain packets. It allows secure token and message passing between Cosmos-Tendermint chains without a third-party bridge (trust \= trusting each chain’s consensus). IBC is extremely trust-minimized and finality depends on the slower of the two chains, typically sub-second to a few seconds.  
* **Polkadot XCMP:** A cross-chain message passing protocol for Polkadot parachains, also using light-client-like proofs. It’s not a bridge for assets but for messages, with trust given by Polkadot’s shared security.  
* **Connext, Celer, Hop (covered above).** (Other bridges like **DeBridge**, **OmniBridge**, etc., use variations of trust-minimized locking or liquidity, but are of smaller scale.)

## ---

**Summary of Findings**

The above protocols vary along several dimensions:

* **Architecture/Model:** Some (Wormhole, Axelar, CCIP, LayerZero) rely on off-chain oracle/signature networks; others (Connext, Hop, Synapse) rely on on-chain liquidity/pool mechanisms; others (Across, Skate) use intent/optimistic models; some (IBC, Polkadot) use inter-chain light clients.  
* **Liquidity:** Bridges like Wormhole, Axelar, CCIP, IBC use *lock/mint* (each chain holds its own locked collateral). In contrast, Connext and Hop use *liquidity pools/routers* that front the transfers. Across mixes both: relayers front from liquidity pools and then are reimbursed. Celer combines both (pool and SGN-backed mint). Skate (hub model) envisions a global state hub to unify liquidity.  
* **Data vs Token:** All support token transfers. Wormhole, CCIP, Axelar, LayerZero, IBC also support arbitrary message passing (to varying extents). Connext and Synapse can pass calldata within EVM context. Across intents can include simple actions (e.g. swap). Hop focuses on tokens.  
* **Speed:** Protocols that use pre-funded liquidity (Connext, Hop, Synapse, Celer’s xLiquidity, Across relayers) offer near-instant completion (seconds to a minute). Lock/mint bridges (Wormhole, Axelar, CCIP, Celer’s xAsset) take longer (seconds-to-minutes) as they await cross-chain consensus/signature. Intent-based approaches (Across, Skate) can be very fast by design (auctions/solvers).  
* **Fees:** Vary widely. Generally, simple bridges charge *gas plus small bridge fee*. Hop/Synapse charge swap fees (\~0.1–0.5%). Cross-chain messaging oracles often add a premium (\~0.05–0.1% on value transferred, plus gas). Intent networks (Across) allow competitive bidding, often resulting in \<1% total.  
* **Security Model:**  
  * *Multi-sig Guardian/Oracle:* Wormhole (guardians), Celer (SGN validators) – require trust in keyholders/validators. Past hacks exist (Wormhole’s 2022 exploit due to a bug).  
  * *Proof-of-Stake Chain:* Axelar uses its own PoS chain (trust \= 2/3 honest validators), with governance controls.  
  * *Decentralized Oracle (DNN):* CCIP uses many node operators and an extra risk-monitoring layer, offering very high security (Chainlink claims “level-5”).  
  * *Non-custodial Pools:* Connext, Hop, Synapse – trust only smart contracts. Vulnerabilities are code bugs. These often have bounty/audits, but LPs bear impermanent loss risk.  
  * *Optimistic Settlements:* Across uses fraud-proof-like checks; trust relies on honest watchers and economic incentives.  
  * *Hybrid (Oracle+Relayer):* LayerZero trusts at least one of the two off-chain services; if both collude, bridge compromised.  
  * *Light Clients:* IBC/Polkadot trust each chain’s finality (very strong, but only works between compatible chains).  
* **Developer/User Experience:** Axelar and CCIP aim for broad developer APIs and single-click integration (Axelar Gateway contracts, Chainlink router libraries). LayerZero endpoints are easy to plug in but require specifying Oracles/Relayers. Connext and Hop have user-friendly UIs/SDKs but limited to certain assets. Across and Skate promise very smooth UX (single intent submission handles the rest).

Overall, there is no one-size-fits-all: bridges like Wormhole/CCIP/Axelar excel at multi-chain reach and generic messaging but incur cross-chain latency and guardian trust. Liquidity bridges like Hop/Connext are very fast and simple but require deep pools on each chain. Intent-based networks like Across/Skate prioritize speed/UX via optimistic economics. Security varies from “code only” (relying on EVM security) to multi-layer oracle networks.

## **Comparative Tables**

| Protocol | Model / Tech | Liquidity Model | Arbitrary Data | Chains Supported | Speed | Trust Assumptions / Security Model |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Wormhole** | Guardian network (19 nodes, t-sigs) | Lock-and-mint (token vaults per chain) | Yes (Token w/Msg) | \~20 (EVM & Solana/Aptos/Sui/etc) | ∼5–15min; new Settlement near-instant | PoA guardians (2/3 threshold), risk of collusion or code bugs |
| **Axelar** | Cosmos PoS hub \+ Gateways (Tendermint \+ TSS) | Lock-and-mint (canonical tokens) | Yes (GMP, Turing-complete) | 50+ (EVMs, Cosmos, Bitcoin…) | \~Minutes (Tendermint \~1s \+ cross-chain latency) | PoS validators (AXL staked, 2/3 honest); has freezing, rate-limits |
| **Across** | Intent+Relayer (optimistic model) | Hub-and-spoke (Spoke pools & central Hub pool) | Limited (user-defined swaps/actions) | EVM L2s (Arbitrum, Optimism, Base, etc.) | Very fast (\<1 min fill) | Optimistic (fraud proof via Oracle); relayers reimburse from Hub pool. Trust \= no fraudulent proofs go unchallenged |
| **Chainlink CCIP** | Decentralized Oracle Networks (DONs) | Lock-and-mint (burn/mint of “CCT” tokens) | Yes (arbitrary bytes) | Wide (EVM, BNB, Solana, etc.) | \~Minutes (waiting oracle consensus \+ txs) | Multi-layer oracles & risk network; very high decentralization, monitors reorgs, etc. |
| **LayerZero** | Modular security (DVNs) \+ independent Executors | Uses app-level pools (no built-in liquidity) | Yes (any message payload, composed messages) | 120+ (EVM, Solana, Move, TON compatible) | Minutes (DVN verification \+ execution) | Modular security owned by application; customizable 'X of Y of N' DVN model; separation of verification and execution; permissionless infrastructure. |
| **Synapse** | AMM pools (bridge tokens) | Pooled liquidity (hTokens per chain) | No (basic calldata only) | \~15 (EVM \+ a few others) | Instant (pool swap) | Smart-contract security only; has had UI hacks, but core trusts EVM security |
| **Connext (NXTP)** | Decentralized liquidity network | Router collateral (lock/unlock) | Yes (supports calldata) | EVM chains \+ L2s | Seconds–minutes (auction \+ cross-tx) | Trustless (no external validators): funds locked in contracts; routers must honor locks or lose funds |
| **Celer (cBridge)** | SGN (Cosmos PoS multi-sig) \+ optional pools | xAsset (lock-mint) & xLiquidity (pools) | No (token transfer only) | 40+ (EVM, BSC, Fantom, etc.) | Fast (instant with pool; seconds otherwise) | SGN consensus (PoS, 2/3 validators honest); vulnerability found but fixed; relies on SGN |
| **Hop** | AMM pools with “hTokens” | Pooled liquidity (hToken/asset pools) | No (token only) | Ethereum \+ major L2s | Instant (pool swap) | Smart-contract security only; no federated trust (but contracts are locked/liquid) |

**Key Differences:** Pools vs vaults vs oracle networks. Protocols like Hop/Connext avoid waiting on consensus by fronting liquidity; bridges like Wormhole/CCIP/Axelar wait for signatures but support more chains. Security ranges from “code only” (relying on EVM security) to fully decentralized oracles (CCIP) to hybrid models. Fees are lowest when liquidity is deep (e.g. Hop) and highest when oracle work is needed. Developer experience also varies: frameworks like CCIP/Axelar provide SDKs, while intent-driven Across/Skate aim to hide complexity from users.

**Pros and Cons Highlights:**

* *Wormhole:* (+) Multi-chain support (including non-EVM), arbitrary messaging; (–) PoA trust, proven exploitable if bugs present.  
* *Axelar:* (+) Unified Cosmos hub, easy token transfers, on-chain governance; (–) depends on its own validator set, bridge fees and finality delay.  
* *Across:* (+) Very fast and cheap bridging, user-friendly intents; (–) relies on optimistic finality (needs oversight) and central Hub liquidity.  
* *CCIP:* (+) Extremely secure (Chainlink’s oracles \+ risk layer), generic messaging; (–) new, fees (oracle premiums), still building out chain coverage.  
* *LayerZero:* (+) Modular security, separation of verification/execution, broad chain support, customizable trust; (–) complexity in DVN configuration for developers.  
* *Connext:* (+) Fully non-custodial, low fees, good for EVM-to-EVM; (–) limited to EVM, requires active router liquidity and potential front-running of auctions.  
* *Celer:* (+) Supports many chains, fast (with pools), hybrid xAsset/xLiquidity; (–) security depends on SGN PoS (vulnerable if many validators hijacked).  
* *Hop:* (+) Fast, simple for ETH/stables between L1/L2; (–) only a handful of tokens, vulnerable to pool liquidity depletion.

Each protocol offers a different trade-off of speed, security, and decentralization.
