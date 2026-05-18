# Original Sin

A 363 MB artwork inscribed onto Ethereum L1 mainnet, by [alphacentaurikid.eth](https://x.com/lphacentaurikid).

Contract: [`0x09A0044F60c48937dcd40f8702D0319469A16a0E`](https://etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E) · Token #1 · 1 of 1

Auction page: [sin.ack.art](https://sin.ack.art)

---

## What's in this repo

This repository contains the public documentation for *Original Sin*, the artist's statement, the technical architecture, the cost analysis, the verification procedure, and a complete reproducible build pipeline for anyone who wants to understand or extend the approach.

- **[`ORIGINAL_SIN.md`](./ORIGINAL_SIN.md)** is the artist's statement, provenance, specification, and frequently asked questions. Start here if you want to understand what the work is.
- **[`ARCHITECTURE.md`](./ARCHITECTURE.md)** is the full technical companion: storage architecture, cryptographic commitment, cost analysis, future-proofing, relationship to prior on-chain art, verification procedure, and a complete terminal walkthrough for reproducing the approach. Start here if you want to understand how it works at the bytecode level, or if you want to build something like it yourself.

---

## TL;DR

*Original Sin* is 363,828,616 bytes (363.83 MB, 15,000 × 11,245 px) stored entirely on Ethereum L1.

The bytes live in event log data across 2,843 sequential mainnet transactions, committed to Ethereum's consensus via the receipts merkle trie. The renderer that reassembles them and displays the image lives in the contract's bytecode via SSTORE2. The standard ERC-721 `tokenURI(1)` returns the full renderer inline as a base64 data URI, so any wallet or marketplace that handles ERC-721 surfaces the artwork without custom integration.

The architecture is determined by physics, not preference: at 363 MB, event logs are the cheapest viable storage mechanism on Ethereum L1 by an order of magnitude. EIP-170 caps contract bytecode at 24,576 bytes per helper contract, ruling out a single-contract `tokenURI` inline approach at this scale. Event logs at ~24 gas/byte (log emission + calldata) carry the same block-header commitment as everything else on the chain.

The total realized cost was approximately **$5,200** in mainnet gas, achieved by timing inscription transactions across overnight low base-fee windows. At typical mid-day mainnet gas (~10 gwei), this architecture would cost in the low six figures for the inscription alone. Gas-window timing is part of the design. The artwork depends on no external storage network, no IPFS, no Arweave, no CDN, no pinning service. Its persistence model is Ethereum's persistence model.

For the full technical breakdown including the per-trie cryptographic commitment analysis, relationship to prior on-chain art (Autoglyphs, deafbeef's generative bytecode work, Ethscriptions inscriptions), and the complete reproducible build pipeline, read [`ARCHITECTURE.md`](./ARCHITECTURE.md).

---

## License

All documentation in this repository is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The architectural patterns are free to use, adapt, and build on with attribution.

The contract source for *Original Sin* is verified on Etherscan: [view source](https://etherscan.io/address/0x09A0044F60c48937dcd40f8702D0319469A16a0E#code).

---

*Original Sin · alphacentaurikid.eth · Ethereum L1 mainnet · 2026*
