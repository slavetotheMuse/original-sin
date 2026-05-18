# ORIGINAL SIN

## A self-reconstructing artwork on Ethereum

By **alphacentaurikid.eth** · 1 of 1 · Ethereum L1 mainnet

[etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E](https://etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E)

---

## What this is

*Original Sin* is a 363 MB digital artwork (15,000 × 11,245 px) stored entirely on Ethereum mainnet.

Not metadata.
Not a thumbnail with a pointer.
Not IPFS.
Not Arweave.
Not a hosted file.

All 363,828,616 bytes of the artwork are inscribed directly into Ethereum transaction logs across 2,843 sequential mainnet transactions. The viewer that reconstructs those bytes into the image is stored inside the same contract's bytecode.

When viewed, the artwork is reconstructed directly from Ethereum archive data in real time. Your browser calls `eth_getLogs`, reassembles the image chunks in memory, verifies the SHA256 hash against the on-chain record, and renders the piece locally.

There is no external file host.
No IPFS gateway.
No Arweave dependency.
No centralized storage layer.

Ethereum itself is the storage layer.

---

## How It Works

The system is built from two on-chain components working together.

### 1. Image bytes, inscribed as event logs

The image was split into 2,843 chunks of 128,000 bytes and written into Ethereum transaction logs through sequential mainnet transactions.

```solidity
event Chunk(bytes32 indexed pieceId, uint256 indexed index, bytes data);
```

Instead of storing the artwork in expensive contract state storage, the bytes live inside Ethereum's historical log data, retrievable through the standard `eth_getLogs` RPC method.

A separate `PieceMeta` event records:

- SHA256 hash
- Image dimensions
- MIME type
- Total byte count

This anchors integrity verification fully on-chain.

### 2. Renderer, stored in contract bytecode

The renderer HTML is stored directly in contract bytecode using the SSTORE2 pattern.

The contract exposes:

```solidity
function tokenURI(uint256 id) external view returns (string memory);
function rendererURL(uint256 id) external view returns (string memory);
```

Both return a complete browser-ready `data:text/html;base64,...` payload.

Pasted into a browser, the renderer reconstructs the artwork directly from Ethereum by:

1. Fetching all chunk events
2. Reassembling the bytes
3. Verifying the SHA256 hash
4. Rendering the image to canvas

The artwork and the viewer exist together as one fully self-contained on-chain system.

*(screen recording of entire process)*

---

## The Method

Many of us ask ourselves: how do you place a truly large, full-resolution artwork on Ethereum without falling back to a pointer?

Not metadata.
Not a hash.
The actual artwork itself.

Working alongside [@richerd](https://x.com/richerd), we explored approaches until he arrived at event-log inscription: storing the artwork's bytes inside Ethereum logs rather than contract state.

That breakthrough made large-scale on-chain storage economically viable.

From there, I expanded the system into a fully self-contained architecture featuring:

- A custom ERC-721 contract
- On-chain renderer storage
- Deep-zoom image reconstruction
- SHA256 verification
- Adaptive scanning
- RPC fallback rotation
- Live byte visualization during loading

The result is an artwork capable of reconstructing itself directly from Ethereum.

---

## Why It Matters

The cost difference is the point.

Cheaper storage systems always depend on infrastructure outside Ethereum:

- AWS depends on Amazon
- IPFS depends on pinning infrastructure
- Arweave depends on the Arweave network

*Original Sin* depends only on Ethereum's archival history.

Its persistence model is Ethereum's persistence model.

If Ethereum survives, the artwork survives.

---

## Viewing the Work

The piece can be viewed multiple ways:

- Through Etherscan using `rendererURL(1)`
- Through a direct browser-rendered on-chain viewer
- Through the ACK viewing interface using optimized RPC infrastructure at [ack.art/originalsin](https://ack.art/originalsin)

In every case, the bytes are reconstructed directly from Ethereum at runtime.

There is no hidden hosted master file elsewhere.

---

## Permanence

The image bytes are immutable.

They cannot be altered, replaced, or removed.

Only the renderer layer remains updateable to maintain compatibility with future browser or RPC infrastructure changes. At the collector's request, even this capability can be permanently renounced.

The artwork itself remains fixed in Ethereum history forever.

---

## Final Thesis

*Original Sin* does not rely on a marketplace.

It does not rely on a website.

It does not rely on external storage infrastructure.

It exists directly within Ethereum's archival history.

The artwork's persistence model is Ethereum's persistence model.

---

## The Auction

The work is currently on auction at [sin.ack.art](https://sin.ack.art), set to end **Tuesday, May 19 at 9 PM Central Time**.

Zero reserve. ETH only.

The winner also receives a signed and framed ChromaSculpt master print at any size up to 120 inches wide, a physical companion to the inscribed edition.

---

*alphacentaurikid.eth*

---

### Further reading

[`ARCHITECTURE.md`](./ARCHITECTURE.md) is the full technical companion: storage architecture, cost analysis, future-proofing, relationship to prior on-chain art, verification procedure, and a complete reproducible build pipeline.
