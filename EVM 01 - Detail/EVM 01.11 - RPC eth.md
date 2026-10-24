---
title: EVM 01.11 - RPC eth
tags: [learning, evm, rpc]
updated: 2026-10-24
---

# RPC eth

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) · [EVM 01.12 - Native Token](EVM%2001.12%20-%20Native%20Token.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.11 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.11%20-%20Hasil%20Praktik.md).

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

Request yang sama persis bisa dikirim ke node Ethereum dan node BSC. Keduanya menjawab dengan
format yang sama; yang berbeda cuma isinya. `eth_chainId` di Ethereum mengembalikan `0x1`
(= 1), di BSC `0x38` (= 56).

Angka saldo dikembalikan dalam hex dan satuan terkecil (wei), jadi perlu dikonversi:
`cast to-dec <hex> | cast from-wei`.

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

Di RPC publik, method `eth_` untuk membaca data menjawab normal, tapi namespace lain sering
dibatasi: `debug_` bisa tidak tersedia sama sekali, `trace_` bisa meminta token khusus untuk
data arsip, dan `engine_` tidak terbuka. Method `rpc_modules` menampilkan namespace yang diaktifkan
sebuah node.

Daftar lengkap dan format tiap method: https://ethereum.github.io/execution-apis/

## Coba sendiri

1. Kirim `eth_chainId` ke node Ethereum dan node BSC, lalu bandingkan.
2. Baca saldo `0x…dEaD` di kedua chain. Di BSC hasilnya dalam BNB.
3. Lihat namespace yang diaktifkan node dengan `rpc_modules`.
4. Coba `eth_accounts` ke RPC publik. Hasilnya `[]`, karena node publik tidak memegang kunci.

```bash
curl -s -X POST https://ethereum-rpc.publicnode.com -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
cast chain-id --rpc-url https://bsc-rpc.publicnode.com
cast balance 0x000000000000000000000000000000000000dEaD --ether --rpc-url <rpc>
cast to-dec <hex> | cast from-wei                  # hex wei → desimal ether
cast rpc rpc_modules --rpc-url <rpc>
cast rpc eth_accounts --rpc-url <rpc>
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) · [EVM 01.12 - Native Token](EVM%2001.12%20-%20Native%20Token.md) →
