---
title: EVM 01.11 - RPC eth
tags: [learning, evm, rpc]
updated: 2026-10-03
---

# RPC eth

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) · [EVM 01.12 - Native Token](EVM%2001.12%20-%20Native%20Token.md) →

Aplikasi (wallet, `cast`, ethers/viem, script) tidak menyentuh blockchain langsung. Ia
bertanya ke sebuah **node** lewat **RPC (Remote Procedure Call)**: kirim HTTP POST berisi
JSON yang menyebut nama method, node membalas hasilnya. Formatnya JSON-RPC 2.0:

```json
{"jsonrpc":"2.0","id":1,"method":"eth_getBalance","params":["0x…dEaD","latest"]}
```

Ethereum menetapkan daftar method standar berawalan `eth_`, misalnya:

| Method | Fungsi |
| --- | --- |
| `eth_chainId` | chain ID |
| `eth_blockNumber` | nomor blok terbaru |
| `eth_getBalance` | saldo native token sebuah alamat |
| `eth_call` | panggil fungsi kontrak tanpa transaksi (mis. `balanceOf`) |
| `eth_sendRawTransaction` | kirim transaksi yang sudah ditandatangani |
| `eth_getLogs` | ambil event |

"Melayani RPC `eth_*` yang sama" artinya node chain itu menerima **nama method, parameter,
dan format jawaban yang sama**. Akibatnya semua tooling Ethereum langsung jalan di chain itu
cukup dengan mengganti URL RPC. Nama `eth_` tetap dipakai walaupun native token-nya bukan
ETH: `eth_getBalance` di BSC mengembalikan saldo **BNB**.

Bukti (2026-10-03), request yang sama persis dikirim ke tiga node:

| Request | Ethereum (`ethereum-rpc.publicnode.com`) | BSC (`bsc-rpc.publicnode.com`) | Solana (`api.mainnet-beta.solana.com`) |
| --- | --- | --- | --- |
| `eth_chainId` | `0x1` (= 1) | `0x38` (= 56) | error `-32601 Method not found` |
| `eth_getBalance` `0xdead` | 12.640,64 ETH | 16.498.949,12 BNB | — |

Solana juga memakai JSON-RPC, tapi daftar method-nya sendiri (`getSlot`, `getBalance`,
`getTokenSupply`, …), jadi tooling EVM tidak bisa dipakai di sana. Angka saldo dikembalikan
dalam hex dan satuan terkecil (wei), jadi perlu dikonversi:
`cast to-dec 0x2ad402176762ad2d59e | cast from-wei` → `12640.64…`.

## Daftar method RPC

Method dikelompokkan per **namespace** (awalan sebelum `_`). Yang standar untuk semua chain EVM
didefinisikan di spesifikasi `ethereum/execution-apis`; sisanya tambahan dari client node
tertentu dan sering dimatikan di RPC publik.

**`eth_` — standar, yang dipakai sehari-hari**

| Kelompok | Method |
| --- | --- |
| Info chain | `eth_chainId`, `eth_blockNumber`, `eth_syncing` |
| Gas / fee | `eth_gasPrice`, `eth_maxPriorityFeePerGas`, `eth_feeHistory`, `eth_blobBaseFee`, `eth_estimateGas` |
| State akun | `eth_getBalance`, `eth_getTransactionCount` (nonce), `eth_getCode`, `eth_getStorageAt`, `eth_getProof` |
| Baca kontrak / simulasi | `eth_call`, `eth_createAccessList`, `eth_simulateV1` |
| Blok | `eth_getBlockByNumber`, `eth_getBlockByHash`, `eth_getBlockReceipts`, `eth_getBlockTransactionCountByNumber`/`ByHash` |
| Transaksi | `eth_getTransactionByHash`, `eth_getTransactionByBlockNumberAndIndex`/`BlockHashAndIndex`, `eth_getTransactionReceipt` |
| Kirim transaksi | `eth_sendRawTransaction` (tx sudah ditandatangani di sisi kamu) |
| Event / log | `eth_getLogs`, `eth_newFilter`, `eth_newBlockFilter`, `eth_newPendingTransactionFilter`, `eth_getFilterChanges`, `eth_getFilterLogs`, `eth_uninstallFilter` |
| Langganan (WebSocket saja) | `eth_subscribe` (`newHeads`, `logs`, `newPendingTransactions`), `eth_unsubscribe` |
| Butuh kunci di node | `eth_accounts`, `eth_sendTransaction`, `eth_sign`, `eth_signTransaction` |

Kelompok terakhir hanya berguna di node lokal yang menyimpan private key (anvil, geth dengan
keystore). Di RPC publik `eth_accounts` kosong dan `eth_sendTransaction` gagal `unknown
account`, karena node publik tidak memegang kunci siapa pun. Itu sebabnya wallet selalu
menandatangani sendiri lalu memakai `eth_sendRawTransaction`.

**Namespace lain**

| Namespace | Isi | Ketersediaan |
| --- | --- | --- |
| `net_` | `net_version`, `net_listening`, `net_peerCount` | standar |
| `web3_` | `web3_clientVersion`, `web3_sha3` | standar |
| `debug_` | `debug_traceTransaction`, `debug_traceCall` (jejak opcode per langkah) | geth/reth/erigon; sering dimatikan atau berbayar |
| `trace_` | `trace_block`, `trace_transaction` (internal call, transfer ETH di dalam kontrak) | erigon/reth/nethermind; biasanya butuh node arsip |
| `txpool_` | `txpool_status`, `txpool_content` (isi mempool) | client tertentu |
| `engine_` | komunikasi execution client ↔ consensus client | **tidak** untuk aplikasi; hanya di port khusus dengan JWT |
| `anvil_` / `hardhat_` / `evm_` | `anvil_impersonateAccount`, `anvil_setBalance`, `anvil_setStorageAt`, `evm_mine`, `evm_snapshot` | hanya node dev lokal (dipakai di [EVM 05 - Fork dan Foundry](../EVM%2005%20-%20Fork%20dan%20Foundry.md)) |

Bukti (2026-10-03), dicoba ke `ethereum-rpc.publicnode.com` (client `Geth/v1.17.1`):
`rpc_modules` → `debug, eth, net, rpc, txpool, …`; semua method `eth_` baca di atas
menjawab; `eth_accounts` → `[]`; `eth_sendTransaction` → `unknown account`;
`txpool_status` → pending `0x10a87` (68.231 tx); `debug_traceCall` → method tidak ada;
`trace_block` → "Archive requests require a personal token"; `engine_exchangeCapabilities`
→ `Method not found`.

Daftar lengkap dan format tiap method: https://ethereum.github.io/execution-apis/

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) · [EVM 01.12 - Native Token](EVM%2001.12%20-%20Native%20Token.md) →
