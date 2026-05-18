# Architecture

A complete technical reference for the storage, retrieval, and verification model of *Original Sin*: a 363 MB artwork inscribed onto Ethereum L1 mainnet.

This document is the technical companion to [`ORIGINAL_SIN.md`](./ORIGINAL_SIN.md). Read that for the artist's statement and provenance. Read this for how it works at the bytecode level, why the architecture is what it is, and how to reproduce the approach.

---

## Contents

1. [The constraint that defined the architecture](#the-constraint-that-defined-the-architecture)
2. [Storage architecture and cryptographic commitment](#storage-architecture-and-cryptographic-commitment)
3. [The hybrid: three layers, three storage locations](#the-hybrid-three-layers-three-storage-locations)
4. [Cost analysis](#cost-analysis)
5. [Future-proofing: RPC fallbacks, EIP-4444, Portal Network](#future-proofing-rpc-fallbacks-eip-4444-portal-network)
6. [Relationship to prior on-chain art](#relationship-to-prior-on-chain-art)
7. [Verification](#verification)
8. [Build it yourself: the full pipeline](#build-it-yourself-the-full-pipeline)
9. [Frequently asked technical questions](#frequently-asked-technical-questions)

---

## The constraint that defined the architecture

*Original Sin* is 363,828,616 bytes. The question that determined every design choice in the contract: **at this size, what storage mechanism on Ethereum L1 is even physically viable?**

The answer is not a matter of preference. The mechanisms available, with their per-byte gas costs and hard structural limits:

| Mechanism | Per-byte gas cost | Hard limit |
|---|---|---|
| `SSTORE` (storage slots) | ~625 gas/byte (amortized; 22,000+ gas per 32-byte slot) | None per contract, but cost scales linearly |
| Contract bytecode via SSTORE2 | ~200 gas/byte | **24,576 bytes per helper contract (EIP-170)** |
| Calldata in self-call transactions | ~16 gas/byte | Per-tx block-gas-limit cap |
| **Event log data** | **~8 gas/byte log + ~16 gas/byte calldata = ~24 gas/byte total** | Per-tx block-gas-limit cap |

For 363 MB at ~24 gas/byte, total gas is ~8.7B. The dollar cost depends entirely on the gas price the deployment runs at:

| Effective gas price | 363 MB via event logs cost (at ETH = $3,000) |
|---|---|
| 10 gwei (typical mid-day mainnet) | ~$262,000 |
| 1 gwei (off-peak overnight) | ~$26,200 |
| **~0.18 gwei (realized in this deployment)** | **~$4,650** |

The deployment was timed across unusually low base-fee windows. The inscription script paused above an 8 gwei threshold and aborted above 20 gwei, so 2,843 transactions only landed during deep overnight troughs. At normal mainnet gas, this architecture would be economically absurd. Patience is part of the design.

There is no "fully in state" version of 363 MB on Ethereum at any gas price. The EIP-170 contract code size limit caps any single contract's bytecode at 24,576 bytes. SSTORE2 inherits that limit and would require thousands of helper contracts to scale, at multiples of the cost. Storage slots are unbounded per contract but at ~625 gas/byte they cost ~30× event logs.

Event log data is the cheapest viable on-chain storage on Ethereum L1 for arbitrary-sized payloads. It also carries the same block-header commitment as everything else on the chain, which is the next section.

---

## Storage architecture and cryptographic commitment

What does it mean for data to be "on" Ethereum? The most precise answer, the one with cryptographic weight, is that the data is **committed to Ethereum's canonical chain via the block header**: anchored in one of the merkle tries whose roots are signed by validators in every block header.

There are four such tries:

| Trie | What it holds | Anchored in block header as |
|---|---|---|
| **State trie** | Account balances, contract storage, contract code | State root |
| **Transactions trie** | All transactions, including their calldata | Transactions root |
| **Receipts trie** | Execution receipts, including all emitted event logs | Receipts root |
| **Withdrawals trie** | Beacon-chain withdrawals (post-Shanghai) | Withdrawals root |

Every block header contains all four roots. Every validator signs the block header. The cryptographic guarantee that "this data is part of Ethereum" applies identically to all four, same signatures, same finality, same chain.

*Original Sin* commits its bytes across two of these tries:

- The **renderer HTML** (~78 KB) lives in the state trie, written as contract bytecode via SSTORE2 helper contracts.
- The **image bytes** (363 MB) live in the receipts trie, written as `Chunk` event log data across 2,843 transactions.

Both are equally anchored. The validator signatures over the state root and the receipts root in any given block header are the same signatures from the same validators on the same block.

### What this means in practice

The 363 MB of image bytes are committed to Ethereum's canonical chain via the receipts root. The renderer is committed via the state root as contract bytecode. The token metadata is committed via the state root as contract code. Ethereum is the storage. There is no IPFS layer, no Arweave layer, no CDN, no centralized host, no pinning service. The artwork persists for as long as Ethereum persists.

---

## The hybrid: three layers, three storage locations

*Original Sin* uses three distinct on-chain storage locations, each chosen because it is the cheapest viable option for what it holds:

### Layer 1: `tokenURI` JSON metadata in contract bytecode

The contract's `tokenURI(1)` function returns a ~107 KB data URI containing the standard ERC-721 metadata JSON. The JSON contains:

- `name`: "Original Sin"
- `description`: artist statement
- `image`: a small SVG placeholder card
- `animation_url`: the full renderer HTML, base64-encoded inline (~78 KB after base64 encoding overhead = ~107 KB total tokenURI size)

The metadata JSON template and the renderer reference are built into the contract's code at deployment time. **Discovery is via the standard ERC-721 interface.** Any wallet, marketplace, or block explorer that handles ERC-721 surfaces the renderer through the same `tokenURI(id)` call it uses for any other NFT. No custom interface is needed.

### Layer 2: Renderer HTML via SSTORE2

The full renderer HTML (the JavaScript program that reassembles the image from event logs and renders it to a canvas with deep-zoom controls) is stored across multiple helper contracts using the **SSTORE2 pattern**:

- The HTML is split into chunks of up to 24,000 bytes each (under EIP-170's 24,576-byte contract code limit).
- Each chunk is deployed as the runtime bytecode of a fresh, minimal helper contract.
- The main contract stores an array of these helpers' addresses.
- Readback is via `EXTCODECOPY` on each helper, concatenating the bytecode.

The renderer is recoverable from the chain via a single contract call (`rendererURL(1)` returns the assembled data URL). It costs ~200 gas/byte to write but is a one-time cost at deploy and is upgradable (the contract owner can replace it to address RPC list staleness, browser API changes, or efficiency improvements). The image bytes themselves are not upgradable. They are immutable event logs.

### Layer 3: Image bytes via event logs

The 363 MB image was split into 2,843 chunks of 128,000 bytes each (with the final chunk shorter) and inscribed via the contract's `inscribe(pieceId, index, data)` and `inscribeBatch(...)` functions. Each call emits a `Chunk` event:

```solidity
event Chunk(bytes32 indexed pieceId, uint256 indexed index, bytes data);
```

The chunk data lives in the transaction log fields, committed to the receipts trie. A separate `PieceMeta` event was emitted once to record the SHA256 hash, MIME type, image dimensions, total byte count, and human-readable signature, anchoring integrity verification on chain.

Cost: ~8 gas/byte for the log data itself, plus ~16 gas/byte for the calldata of the inscribing transaction. **Total: ~24 gas per byte stored, vs ~625 gas/byte for direct storage slots**. The 25× cost difference is what makes 363 MB economically viable.

### Why the renderer is "in the chain" too

A subtle but important property of the architecture: the renderer is not loaded from a server, an IPFS gateway, or any external host. It is encoded into the `animation_url` field of `tokenURI`, base64-decoded, and executed in the browser. The bootstrap is fully self-contained:

1. Browser fetches `tokenURI(1)` from any RPC endpoint → gets the JSON with the inline renderer.
2. Browser opens the `animation_url` → renderer HTML runs.
3. Renderer calls `eth_getLogs` on any RPC endpoint → fetches the image chunks.
4. Renderer concatenates, verifies SHA256, paints to canvas.

No domain name needs to exist. No website needs to be hosted. No CDN, IPFS pin, Arweave bundle, or pinning service is needed at any step. The only off-chain dependency is the RPC endpoint used for the eth_getLogs calls, which is also true of every other Ethereum application, and is mitigated by the renderer's built-in fallback list and ability to use any provider including the viewer's own wallet provider.

---

## Cost analysis

### Realized deployment cost

| Operation | Realized cost (ETH ~$3,000) |
|---|---|
| Contract deploy + Etherscan verify | ~$50 |
| Renderer: 5 SSTORE2 chunks (~24 KB each) | ~$60 |
| Image inscription: 2,843 chunks × 128,000 bytes = 363,828,616 bytes total | ~$4,650 |
| Mint to Trezor + ownership transfers | ~$5 |
| Renderer upgrades (RPC list, performance) | ~$400 |
| Other transactions (testing, iteration) | ~$130 |
| **Total** | **~$5,200** |

### Why this is far below the typical-gas-price cost

At normal mid-day mainnet gas prices (~10 gwei), this architecture would cost in the low six figures for the image inscription alone. The deployment ran across overnight low-gas windows, with the inscription script pausing above an 8 gwei threshold and aborting above 20 gwei. Almost all of the 2,843 inscribing transactions landed during deep base-fee troughs, averaging well under one gwei effective. Gas-window timing is a deliberate part of the architecture; patience converts directly to cost reduction at this scale.

### Per-byte cost comparison across mechanisms

For 363 MB at 10 gwei, ETH = $3,000:

| Mechanism | Gas/byte | 363 MB cost at 10 gwei | Notes |
|---|---|---|---|
| `SSTORE` direct | ~625 | ~$6.8M | unbounded per contract, scales linearly |
| Contract bytecode (SSTORE2) | ~200 | ~$2.18M | bounded by EIP-170 to 24 KB per helper contract |
| Calldata in TX | ~16 | ~$174k | per-tx block gas limit applies |
| **Event log data (this work)** | **~24 (log + calldata)** | **~$262k** | per-tx block gas limit applies |

Event logs are the cheapest viable mechanism at any gas price. The realized deployment cost was an order of magnitude lower than the 10-gwei figure because of gas-window timing, not architectural difference.
| IPFS (paid pinning) | n/a | ~$20/year | Pinning service operations |
| Arweave | n/a | ~$0.75 one-time | Arweave network |
| AWS S3 Standard | n/a | ~$0.10/year | AWS + active subscription |

The gap between any off-chain option ($0.10–$20) and Ethereum L1 ($4,650) is the entire artistic position. Off-chain options depend on infrastructure other than Ethereum. This work depends on nothing but Ethereum.

---

## Future-proofing: RPC fallbacks, EIP-4444, Portal Network

### RPC fallback design

The renderer does not depend on any single RPC provider. Its resolution order:

1. **`window.ethereum`.** If the viewer is in a wallet-injected context (browser extension, marketplace iframe with wallet support), the wallet's RPC provider is used. No external dependency.
2. **`?rpc=` URL query parameter.** Viewer can pass their own RPC endpoint, including their own self-hosted node.
3. **Hardcoded fallback list.** A rotating set of public mainnet RPCs (publicnode, tenderly, mevblocker), with adaptive block-window halving on errors.

The fallback list is specifically composed of multi-org public infrastructure that doesn't depend on any single company's continued operation. If one provider rate-limits or goes offline, the renderer rotates to the next. If all three are simultaneously unavailable and the viewer has no wallet and provides no `?rpc=` override, the artwork doesn't render. The same is true of every Ethereum application.

The fallback list itself is updatable. The contract owner can call `replaceRenderer` to ship a new version with refreshed RPC endpoints if the current list becomes stale. This is a five-minute, ~$130–$200 operation. The image bytes are not affected by renderer updates.

### EIP-4444 and Ethereum's statelessness roadmap

Ethereum's statelessness roadmap concerns the **state trie**, enabling clients to verify blocks without holding the full current state, using witnesses. It does not propose dropping event logs, transaction receipts, or any historical data from chain.

**EIP-4444** is the closest thing to a log-retention concern. It proposes that clients stop serving historical headers, bodies, and receipts older than roughly one year over the p2p layer, with clients permitted to locally prune that historical data. Initial partial-history-expiry work has focused on pre-Merge history, but the broader EIP-4444 model is rolling history expiry. The EIP explicitly requires archive infrastructure to be in place before activation. Archive nodes continue to retain everything, and the **Portal Network** is being built as a decentralized archival layer that distributes historical-data responsibility across many nodes rather than concentrating it on a few archive providers.

Importantly, **EIP-4444 does not affect the cryptographic commitment of the data.** The bytes remain anchored in every block header's receipts root. EIP-4444 changes which nodes serve the data by default, not whether the data is part of the chain. The same applies to historical contract storage that's been written but isn't actively read. Long-term archival of any historical Ethereum data (state, logs, transactions, anything older than the "warm" retention window) depends on archive providers and the Portal Network. All of it carries the same archival assumption.

In practical terms: if Ethereum's historical data remains broadly accessible (the condition under which any on-chain claim holds long-term, including pieces stored in contract state slots), *Original Sin* remains accessible. The artwork's persistence model is Ethereum's persistence model. It adds no separate archival assumption.

---

## Relationship to prior on-chain art

Placing bytes on Ethereum is not new. *Original Sin* sits in a lineage of fully on-chain art that has explored different parts of the design space. Each prior tradition solved a different problem with a different constraint:

### Inline tokenURI work

The earliest fully on-chain approach: a contract's `tokenURI()` function returns a `data:` URL containing the image itself, base64-encoded inline. Larva Labs's *Autoglyphs* (2019, ~5 KB SVGs per token) and Richerd Chan's experiments are canonical examples. One contract call returns the complete bytes. The bytes are read directly from contract code or state. No external storage of any kind.

This is the tightest possible self-containment. Its constraint is scale: a 363 MB image cannot exist in any contract's `tokenURI` return value, because the EIP-170 contract code size limit caps a single contract's deployed code at 24,576 bytes, and SSTORE2 doesn't lift that constraint without an unwieldy tree of helper contracts. Inline tokenURI is the right architecture for small pieces (typically under ~20 KB).

### Generative bytecode work

This isn't the same technical problem as the generative bytecode work pioneered by artists like **@_deafbeef**.

Those systems are fundamentally about compact executable logic. The artwork exists primarily as code: algorithms, shaders, SVG instructions, deterministic generation rules, compressed runtime behavior. The emphasis is elegance, compression, composability, and EVM-native execution.

*Original Sin* solves a different problem.

We were not trying to generate an image from compact instructions. We were trying to embed a 363 MB full-resolution source image itself directly into Ethereum's canonical chain without external storage.

That changes everything architecturally.

**Generative bytecode systems optimize for:**

- minimal storage footprint
- deterministic generation
- executable contract logic
- compactness and elegance
- EVM composability

***Original Sin* optimizes for:**

- massive binary permanence
- archival-scale data inscription
- preservation of the exact source image
- Ethereum-native reconstruction
- retrieval directly from historical log data

These are not interchangeable architectures.

One is closer to a program that creates an artwork at runtime. The other is closer to engraving the artwork itself directly into Ethereum's historical record.

Both are valid. Both are important. They solve fundamentally different technical and artistic problems.

### Inscription work

The **Ethscriptions** pattern (2023) and related work embed arbitrary bytes in the calldata of plain ETH transactions. The bytes live in the transactions trie rather than the state or receipts trie, anchored cryptographically the same way. This approach trades the lack of any contract logic (no view functions, no on-chain rendering interface) for simplicity and even lower cost.

*Original Sin* differs from this lineage in two ways: it uses event logs (committed via receipts trie, emitted from contract execution) rather than calldata (transactions trie, no contract execution required), and it ships with an on-chain renderer that surfaces the assembled artwork through the standard ERC-721 interface. The renderer makes the inscription viewable as an NFT in any standard wallet or marketplace, not just as raw bytes for someone to decode.

### Where *Original Sin* sits

*Original Sin* solves a problem none of the above categories were designed to solve: **placing a 363 MB full-resolution source image on Ethereum L1, recoverable from chain alone, with a self-contained on-chain viewer surfaced through the standard ERC-721 discovery interface**.

It uses event logs because they are the only storage mechanism on Ethereum L1 that can hold 363 MB at viable cost while remaining anchored to consensus. It uses contract bytecode (SSTORE2) for the renderer because the renderer is small enough to fit and benefits from being readable via a normal contract call. It uses `tokenURI` to surface the renderer through the standard ERC-721 surface, so any wallet or marketplace that handles ERC-721 handles this token without custom integration.

The architecture is the only one available at this size. None of the prior categories were built for storing source images at this scale, and none of them claim to be. *Original Sin* sits in a different constraint regime than the prior categories cover, not by competing with them on their own terms, but by occupying a problem space they don't.

---

## Verification

Anyone, at any time, can independently verify the artwork's existence and integrity:

### Step 1: Read the on-chain hash

```bash
# Via Etherscan:
# https://etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E#readContract
# Call pieceHash(0x0275dcbf…6d530)
# Returns: 0x09341951…16ffdb

# Or via RPC:
cast call 0x09A0044F60c48937dcd40f8702D0319469A16a0E \
  "pieceHash(bytes32)(bytes32)" 0x0275dcbf…6d530 \
  --rpc-url https://ethereum-rpc.publicnode.com
```

### Step 2: Fetch all chunks

```bash
# Query eth_getLogs for the contract's Chunk events with this pieceId.
# The Chunk event signature:
#   Chunk(bytes32 indexed pieceId, uint256 indexed index, bytes data)
# Filter: topic[0]=keccak256("Chunk(bytes32,uint256,bytes)"), topic[1]=pieceId
# Result: 2,843 logs containing 128,000 bytes each (last chunk shorter)
```

### Step 3: Reassemble + hash

```bash
# Concatenate the chunks in order of their `index` topic (topic[2]).
# Compute SHA256 of the concatenated bytes.
# Result must match the hash from Step 1.
```

### Step 4: Decode

```bash
# The result is a valid PNG file. Open it in any image viewer.
# Width: 15,000 px. Height: 11,245 px. Size: 363,828,616 bytes.
```

A complete reference implementation of this verification is the renderer itself, readable via `rendererURL(1)` on the contract. The renderer does exactly these steps in JavaScript: fetches the chunks, reassembles them, verifies the hash, and renders the image. Anyone can replicate this in any language.

---

## Build it yourself: the full pipeline

For artists, developers, or technical collectors who want to understand or reproduce the approach, here is the full pipeline as it actually ran.

### Prerequisites

- Node.js 20+, git, Hardhat
- A funded Ethereum mainnet wallet (Trezor recommended for deploy and ownership transactions)
- An Ethereum RPC endpoint with reliable mainnet access (Alchemy, Infura, your own node, or a paid provider for the inscription stage, public endpoints rate-limit at the volume needed)
- An Etherscan API key for contract verification
- Budget: ~$5,000–$10,000 in mainnet gas for a 363 MB file at 5–25 gwei
- Sepolia testnet wallet for a full dress rehearsal (free; required)

### 1. Prepare the artwork

```bash
shasum -a 256 your-artwork.png
# Records the SHA256, this becomes the on-chain identity of the piece

identify -format "%w x %h, %B bytes\n" your-artwork.png
# Confirms dimensions and exact byte count
```

### 2. Project setup

```bash
git clone <your-fork-or-template>
cd onchain
npm install
cp .env.example .env
# Add PRIVATE_KEY, RPC_URL, ETHERSCAN_API_KEY
```

Key source files:

- `contracts/Quantam.sol`: ERC-721 with inscription events + SSTORE2 renderer storage
- `viewer/onchain-renderer.html`: self-contained bootloader that reassembles bytes from logs

### 3. Compile

```bash
npx hardhat compile
```

### 4. Dress rehearsal on Sepolia

Deploy on Sepolia, run the entire pipeline with a tiny test file, verify rendering before touching mainnet:

```bash
npx hardhat run scripts/deploy.js --network sepolia
# ✅ Quantam deployed to: 0x...

export CONTRACT=0x...
export FILE=./small-test-file.jpg
export MIME=image/jpeg WIDTH=1500 HEIGHT=1124
export TITLE=Test SIGNATURE="dress rehearsal"
export TOKEN_ID=1 MINT_TO=0xYourTestAddress
node scripts/upload.js
```

This exercises everything end-to-end. Cost: free (Sepolia faucet).

### 5. Mainnet deploy

```bash
./scripts/wait-for-gas.sh 5
# Waits until base fee ≤ 5 gwei

npx hardhat run scripts/deploy.js --network mainnet
# ✅ Quantam deployed to: 0x...
```

Cost: ~$50–$150 depending on gas.

### 6. Inscribe the renderer (stage 1)

```bash
export CONTRACT=<deployed-contract-address>
export FILE=./your-artwork.png
export MIME=image/png WIDTH=15000 HEIGHT=11245
export TITLE="Your Piece Title"
export SIGNATURE="Your artist statement"
export TOKEN_ID=1 MINT_TO=<your-trezor-address>
export GAS_PRICE_GWEI=8        # pause if base fee exceeds this
export MAX_GAS_PRICE_GWEI=20   # abort if base fee spikes above this

# Preview costs before signing anything:
DRY_RUN=1 node scripts/upload.js

# Real run:
node scripts/upload.js
# Stage 1: appending renderer chunk 1/4 ...
# Stage 1: appending renderer chunk 2/4 ...
```

Cost: ~$30–$80 depending on renderer size and gas.

### 7. Inscribe the image (stage 2, the long one)

The same `upload.js` continues automatically into stage 2 after stage 1 finishes. It queries `chunkCount(pieceId)` to determine progress, so it's fully resumable. Safe to ctrl-C and rerun.

```bash
node scripts/upload.js
# Stage 2: emitted meta event ✓
# Stage 2: chunk 1/2843 ...
# Stage 2: chunk 2/2843 ...
# ...
# (4–12 hours depending on gas conditions and chunk size)
```

Each chunk is one mainnet transaction. The script:

- Pauses when base fee exceeds `GAS_PRICE_GWEI`
- Aborts if base fee exceeds `MAX_GAS_PRICE_GWEI`
- Uses `inscribeBatch` to pack multiple chunks per tx where block gas limits allow
- Is fully resumable, kill and restart anytime; it picks up from `chunkCount(pieceId)`

This is the largest gas line item. Budget ~$4,000–6,000 at typical mainnet conditions.

### 8. Mint the token (stage 3)

Runs automatically at the end of `upload.js`:

```
Stage 3: mint(<address>, pieceId, "Your Piece Title", "Your description") → tokenId 1
```

After mint, `tokenURI(1)` returns a JSON data URI containing `name`, `description`, `image` (small SVG placeholder), and `animation_url` (the base64-encoded renderer HTML).

### 9. Verify

```bash
node scripts/verify-onchain.js
# Fetching 2843 chunk events from chain...
# Reassembling bytes by chunk index...
# Computing SHA256...
# Result: 0x09341951…16ffdb
# Expected: 0x09341951…16ffdb (from PieceMeta event)
# ✓ Match, on-chain bytes perfectly reconstruct original file
```

This proves the bytes on chain reconstruct the file exactly.

### 10. View

```bash
cast call $CONTRACT "rendererURL(uint256)" 1 \
  --rpc-url $RPC_URL | head -c 200
# data:text/html;base64,PHNjcmlwdD53aW5kb3cuUVVBTlRBTV9DT05GSUc...

# Paste the full output into any browser address bar
# Browser fetches log events from chain, reassembles bytes, renders image
```

Three equivalent viewing paths:

- **Etherscan → Read Contract → `rendererURL(1)` → copy → paste in Chrome.** Purest path. No website, no marketplace. Etherscan and browser only.
- **Any wallet or marketplace that respects `animation_url`.** Standard ERC-721 surface; works in OpenSea, Manifold, Zora, etc.
- **A hosted viewer site.** Same data, friendlier UX. The auction page at sin.ack.art is this path.

### Summary of layers

| Layer | What | Storage | Cost per byte |
|---|---|---|---|
| tokenURI JSON | Metadata + base64-encoded renderer | Contract bytecode (state trie) | ~200 gas/byte |
| Renderer HTML | Self-contained viewer | SSTORE2 chunks in helper contract bytecode (state trie) | ~200 gas/byte |
| Image bytes | The 363 MB source image | Event log data (receipts trie) | ~24 gas/byte total |
| Reassembly | Concatenation, hash check, render | Browser-side via `eth_getLogs` | n/a (runtime) |

Each layer uses the cheapest Ethereum-native storage that can hold its content. All four are anchored in every Ethereum block header.

---

## Frequently asked technical questions

### Why event logs?

Two reasons:

1. **Cost.** Event log data is ~80× cheaper per byte than `SSTORE`-based storage slots and ~25× cheaper than SSTORE2 contract-bytecode storage. At 363 MB, this is the difference between $4,650 and $200,000+.
2. **Physical capacity.** SSTORE2 inherits EIP-170's 24,576-byte per-contract-bytecode cap. There is no way to fit 363 MB in a single contract's bytecode, and the "tree of helper contracts" approach gets unwieldy and expensive past a few MB. Event logs have no per-piece capacity limit beyond the per-transaction block gas limit, which the inscription script handles by chunking across many transactions.

Event log data and contract storage are committed to Ethereum's consensus identically: both anchored via merkle roots signed by every validator in every block header. For storing 363 MB of image bytes, event logs are simply the right tool.

### What if all public RPCs in the renderer's fallback list go offline?

The renderer resolves the RPC in this order:

1. `window.ethereum` (wallet-injected provider, covers most viewing contexts)
2. `?rpc=` URL parameter (viewer's own RPC)
3. Hardcoded fallback list: publicnode, tenderly, mevblocker (three independent organizations)

For all three fallbacks to be simultaneously unavailable AND the viewer to have no wallet AND not pass a `?rpc=` parameter is a low-probability event. If it occurs, the renderer can be updated (~$130–$200, five-minute operation) to ship a fresh fallback list. The image bytes are immutable event logs and are not affected.

For a viewer running their own Ethereum node, the artwork renders directly from their own node with no third-party dependency at all.

### What about EIP-4444 / history expiry?

EIP-4444 proposes that clients stop serving historical headers, bodies, and receipts older than roughly one year over the p2p layer, with clients permitted to locally prune that historical data. The broader model is rolling history expiry. The EIP requires archive infrastructure (archive nodes + Portal Network) to be in place before activation, and archive providers and Portal Network nodes are designed to retain everything indefinitely.

EIP-4444 does not change the block-header commitment of the data. The receipts root remains in every block header forever. The data is part of Ethereum's canonical chain regardless of which nodes happen to serve it at any given moment.

In practice: any long-term Ethereum data access, including historical contract storage that isn't actively read, depends on archive infrastructure. EIP-4444 doesn't introduce a new dependency for event-log-based pieces; it formalizes the dependency that already exists for any historical Ethereum data.

### Can the artwork be tampered with via a malicious renderer?

The contract's renderer is updatable by the contract owner. A malicious renderer could in principle render different bytes than the inscribed ones. The integrity guarantee against this is **SHA256 verification at view time**: the renderer reassembles the inscribed bytes from logs, computes SHA256, and compares against the immutable on-chain `PieceMeta` hash. A malicious renderer that skipped this check would render the wrong image, which any verifier would catch.

For the strictest guarantee, the buyer can request, and the artist has committed to honoring, that contract ownership be transferred to `0x0` after the sale. This makes the contract permanently immutable. Until that transfer, the artist holds the owner role specifically to maintain the renderer (fallback RPC updates, browser API changes), not the bytes.

### Why include the renderer on chain rather than just inscribing the bytes?

The renderer being on chain (in contract bytecode, surfaced via `tokenURI`) means the artwork is viewable through the standard ERC-721 interface that every wallet, marketplace, and explorer already speaks. Any system that handles ERC-721 handles this token. No custom indexer, no special viewer, no off-chain renderer host is needed for the viewing experience.

Without the on-chain renderer, the inscribed bytes would still be retrievable from chain, but viewing them would require someone to either (a) write their own renderer from scratch, or (b) trust a third-party hosted viewer. Both work, but neither is the bar this work is aiming at. The on-chain renderer makes the artwork self-contained from chain to canvas.

### How is this different from Ethscriptions?

Ethscriptions embed bytes in **transaction calldata** (the data sent to a transaction) rather than event logs. Calldata is committed to the **transactions trie**, anchored in the block header just like the state trie and receipts trie. Both Ethscriptions and *Original Sin* are equally on chain in the consensus-commitment sense.

The two differ in:

1. **Storage trie**: calldata vs receipts trie. Same consensus weight, different retrieval path.
2. **Contract execution**: Ethscriptions are inscribed in calldata of plain ETH transfers (no contract called). *Original Sin* is inscribed via contract function calls that emit events. The contract execution is what enables the on-chain renderer and ERC-721 interface.
3. **Viewing surface**: Ethscriptions have no built-in viewing interface from any single contract call; you query an indexer or write your own decoder. *Original Sin* exposes the full assembled renderer via standard `tokenURI(1)`.

The two architectures are appropriate for different goals. Ethscriptions optimize for cheapest possible inscription. *Original Sin* optimizes for a complete on-chain viewing experience surfaced through the standard NFT interface, scaled up to where event-log storage becomes the only viable option.

### What's the actual gas math for 24 gas/byte?

Event log data on Ethereum L1 costs:

- **8 gas per byte of log data** (the `LOG{0-4}` opcode pricing)
- **16 gas per non-zero byte of calldata** (for the data the contract receives in the inscribing transaction; the contract then re-emits it as log data)
- ~4 gas per zero byte of calldata

For a typical 128 KB image chunk, calldata is mostly non-zero (compressed image data is high-entropy), so the dominant cost is ~16 + 8 = ~24 gas per byte. Plus base transaction cost (21,000 gas) and per-log overhead (~375 gas + 375 gas per indexed topic), amortized over the chunk size.

Total per-byte cost at 128 KB chunks ≈ 24 gas/byte, putting 363 MB at roughly:

```
363,828,616 bytes × 24 gas/byte = ~8.7B gas total
At 10 gwei × $3,000/ETH = ~$260,000 hypothetical cost at full block load
At realized gas prices (deploy windows targeted low base fees) ≈ ~$4,650 actual
```

The script's gas-gating (pause above 8 gwei, abort above 20 gwei) is what kept the realized cost below half a percent of the worst-case mainnet pricing. Patience is one of the architectural choices.

---

## License

This document is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The architectural patterns described are free to use, adapt, and build on with attribution.

The contract source for *Original Sin* is verified on Etherscan: [0x09A0044F60c48937dcd40f8702D0319469A16a0E](https://etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E#code).

---

*Original Sin · alphacentaurikid.eth · Ethereum L1 mainnet · 2026*
