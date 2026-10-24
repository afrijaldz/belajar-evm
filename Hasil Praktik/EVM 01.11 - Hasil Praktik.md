---
title: EVM 01.11 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.11 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.11 - RPC eth](../EVM%2001%20-%20Detail/EVM%2001.11%20-%20RPC%20eth.md). Dijalankan 2026-10-03 ke RPC publik
publicnode: `ethereum-rpc.publicnode.com` dan `bsc-rpc.publicnode.com`.

## Request yang sama ke dua chain

Request yang sama persis dikirim ke dua node:

| Request | Ethereum (`ethereum-rpc.publicnode.com`) | BSC (`bsc-rpc.publicnode.com`) |
| --- | --- | --- |
| `eth_chainId` | `0x1` (= 1) | `0x38` (= 56) |
| `eth_getBalance` `0xdead` | 12.640,64 ETH | 16.498.949,12 BNB |

Konversi saldo Ethereum dari hex wei:
`cast to-dec 0x2ad402176762ad2d59e | cast from-wei` → `12640.64…`.

## Namespace di RPC publik

Dicoba ke `ethereum-rpc.publicnode.com` (client `Geth/v1.17.1`):

| Request | Hasil |
| --- | --- |
| `rpc_modules` | `debug, eth, net, rpc, txpool, …` |
| semua method `eth_` baca di tabel materi | menjawab |
| `eth_accounts` | `[]` |
| `eth_sendTransaction` | `unknown account` |
| `txpool_status` | pending `0x10a87` (68.231 tx) |
| `debug_traceCall` | method tidak ada |
| `trace_block` | "Archive requests require a personal token" |
| `engine_exchangeCapabilities` | `Method not found` |
