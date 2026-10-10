---
title: EVM 01 - EVM vs Non-EVM
tags: [learning, ethereum, evm, bsc, solana, hyperliquid]
updated: 2026-10-10
---

# EVM vs Non-EVM

Pertanyaan: apa itu EVM, dan kapan sebuah chain disebut chain EVM?

**EVM adalah mesin eksekusinya, bukan koin atau chain tertentu.** Ethereum, BSC, Polygon,
Base, Arbitrum, dan Avalanche C-Chain semuanya chain EVM, walaupun native token-nya berbeda.

## EVM itu mesinnya, bukan koinnya

**EVM (Ethereum Virtual Machine)** adalah mesin eksekusi — "komputer virtual" yang menjalankan
smart contract. Sebuah chain disebut EVM kalau ia menjalankan bytecode yang sama, memakai
format alamat `0x…` yang sama, dan melayani RPC `eth_*` yang sama.

### Penjelasan detail

Setiap bagian definisi di atas dijelaskan di file terpisah, folder `EVM 01 - Detail/`. Baca berurutan:

1. [EVM 01.01 - Mesin EVM](EVM%2001%20-%20Detail/EVM%2001.01%20-%20Mesin%20EVM.md) — Gambaran EVM sebagai komputer virtual: posisinya di node, isi mesinnya, dan contoh program jalan langkah demi langkah
2. [EVM 01.02 - Opcode](EVM%2001%20-%20Detail/EVM%2001.02%20-%20Opcode.md) — Opcode = satu perintah EVM berukuran 1 byte
3. [EVM 01.03 - Stack](EVM%2001%20-%20Detail/EVM%2001.03%20-%20Stack.md) — Stack: maksimal 1024 kotak, tiap kotak 32 byte
4. [EVM 01.04 - Memory dan Calldata](EVM%2001%20-%20Detail/EVM%2001.04%20-%20Memory%20dan%20Calldata.md) — Memory (coretan sementara) vs calldata (input read-only)
5. [EVM 01.05 - Environment](EVM%2001%20-%20Detail/EVM%2001.05%20-%20Environment.md) — Environment: info situasi eksekusi (msg.sender, tx.origin, block.*)
6. [EVM 01.06 - Storage](EVM%2001%20-%20Detail/EVM%2001.06%20-%20Storage.md) — Storage: penyimpanan permanen kontrak, 2²⁵⁶ slot × 32 byte
7. [EVM 01.07 - Program Counter](EVM%2001%20-%20Detail/EVM%2001.07%20-%20Program%20Counter.md) — PC: penunjuk byte yang sedang dijalankan, JUMP/JUMPI/JUMPDEST
8. [EVM 01.08 - Gas](EVM%2001%20-%20Detail/EVM%2001.08%20-%20Gas.md) — Gas: satuan kerja, gas limit/used/price, OutOfGas
9. [EVM 01.09 - Bytecode yang Sama](EVM%2001%20-%20Detail/EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) — Arti "menjalankan bytecode yang sama"
10. [EVM 01.10 - Format Alamat 0x](EVM%2001%20-%20Detail/EVM%2001.10%20-%20Format%20Alamat%200x.md) — Arti "format alamat 0x… yang sama"
11. [EVM 01.11 - RPC eth](EVM%2001%20-%20Detail/EVM%2001.11%20-%20RPC%20eth.md) — Arti "melayani RPC eth_* yang sama" + daftar method RPC
12. [EVM 01.12 - Native Token](EVM%2001%20-%20Detail/EVM%2001.12%20-%20Native%20Token.md) — Native token vs token ERC-20: disimpan di mana, cara kirim/cek, WETH
13. [EVM 01.13 - EVM, Node, dan Validator](EVM%2001%20-%20Detail/EVM%2001.13%20-%20EVM%2C%20Node%2C%20dan%20Validator.md) — EVM bukan node atau validator: EVM ada di dalam node, validator adalah peran
14. [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001%20-%20Detail/EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md) — Execution client (isi blok, EVM, state) vs consensus client (blok mana yang sah, PoS), Engine API
15. [EVM 01.15 - Validator Client](EVM%2001%20-%20Detail/EVM%2001.15%20-%20Validator%20Client.md) — Validator client: program terpisah pemegang kunci, terhubung ke beacon node, slashing, dua jenis kunci
16. [EVM 01.16 - Slashing](EVM%2001%20-%20Detail/EVM%2001.16%20-%20Slashing.md) — Slashing: tiga pelanggaran, alur hukuman, penalti korelasi, kasus 17 validator double vote (Agustus 2026)
17. [EVM 01.17 - Attestation](EVM%2001%20-%20Detail/EVM%2001.17%20-%20Attestation.md) — Attestation: isi suara, komite per slot, agregasi BLS, fork choice dan finality
18. [EVM 01.18 - Finality](EVM%2001%20-%20Detail/EVM%2001.18%20-%20Finality.md) — Finality: checkpoint, justified, finalized, kenapa 2/3, tag `safe`/`finalized`, fast finality BSC

### Native token

**Native token** cuma mata uang untuk membayar gas di chain itu. Setiap chain bebas memilih.
Penjelasan detail: [EVM 01.12 - Native Token](EVM%2001%20-%20Detail/EVM%2001.12%20-%20Native%20Token.md).

| Chain | Mesin | Native token (gas) |
| --- | --- | --- |
| Ethereum | EVM | ETH |
| **BSC** | **EVM** | **BNB** |
| Polygon PoS | EVM | POL |
| Avalanche C-Chain | EVM | AVAX |
| HyperEVM | EVM | HYPE |
| Solana | **SVM** (non-EVM) | SOL |
| Bitcoin | Script (non-EVM) | BTC |
| Sui, Aptos | **Move VM** (non-EVM) | SUI, APT |

Analogi: EVM itu seperti Android. Samsung dan Xiaomi sama-sama Android walaupun mereknya
berbeda, dan aplikasi yang sama bisa berjalan di keduanya.

## Bukti: perintah yang sama di chain berbeda (2026-09-24)

Membaca token di BSC (CAKE Diperiksa Ulang 2026-09-24, BNB Diperiksa Ulang 2026-09-24)
memakai perintah yang **persis sama** dengan membaca token di Ethereum
(UNI Diperiksa dengan Saringan Lima Chain, ETHFI Diperiksa dengan Saringan Lima Chain):

- `eth_call` dengan selector `0x70a08231` (`balanceOf`) dan `0x18160ddd` (`totalSupply`)
- `eth_getBalance` untuk membaca BNB di `0xdead`
- `cast` (Foundry) pindah chain cukup dengan mengganti `--rpc-url`
- alamat `0x000000000000000000000000000000000000dEaD` berlaku di kedua chain

Hal yang sama berlaku di chain EVM lain: Base dan Arbitrum dibaca dengan cara yang sama di
[EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md).

Solana berbeda. Di sana dipakai `getTokenSupply` dan `getTokenAccountsByOwner` — API yang
lain sama sekali — karena Solana **bukan EVM**. Model datanya juga beda: satu wallet bisa punya
banyak token account (wallet buyback RAY punya 6). Lihat Account Model Solana dan
RAY JUP ORCA Diperiksa Ulang 2026-09-24.

## Hal yang sering membingungkan

- **Native token tidak menentukan EVM atau bukan.** BSC memakai BNB dan Polygon memakai POL
  untuk gas, tapi keduanya tetap chain EVM.
- **Nama standar token bisa beda per chain, isinya sama.** BEP-20 di BSC = ERC-20: interface
  identik, cuma namanya diganti (Binance Evolution Proposal). Karena itu kontrak CAKE punya
  `balanceOf`, `transfer`, dan seterusnya, sama seperti token ERC-20 di Ethereum.
- **Token bernama "ETH" di chain lain biasanya bukan ETH asli.** Contohnya "ETH" di BSC: token
  BEP-20 (Binance-Peg ETH) yang dijamin Binance. Di sana ETH cuma token biasa; gas tetap
  dibayar dengan BNB.
- **Kalau mesinnya sama, bedanya di mana?** Di lapisan lain. Contoh Ethereum vs BSC:

  | | Ethereum | BSC |
  | --- | --- | --- |
  | Konsensus | PoS, validator sangat banyak | PoSA, validator jauh lebih sedikit (lebih cepat, lebih terpusat) |
  | Block time | 12 detik | **0,45 detik** (terukur 2026-09-24) |
  | Aturan ekonomi khusus | EIP-1559 burn | BEP-95 burn + auto-burn kuartalan |
  | Chain ID | 1 | 56 |

  Chain ID dipakai wallet untuk membedakan chain yang mesinnya sama.
- **Kode node:** banyak chain EVM memakai ulang client Ethereum. BSC adalah fork dari
  **go-ethereum (geth)**, jadi secara teknis BSC adalah "Ethereum yang dimodifikasi".
- **Hyperliquid punya dua bagian:**
  - **HyperCore** — orderbook perp, **non-EVM**, diakses lewat `api.hyperliquid.xyz`
  - **HyperEVM** — EVM, gas pakai HYPE

  Karena itu supply HYPE dibaca dari API khusus (`tokenDetails`), bukan `eth_call`. Lihat
  HYPE Diperiksa Ulang 2026-09-24.

## Urutan belajar berikutnya

1. Model akun EVM: EOA vs contract account
2. Cara transaksi dan gas bekerja di EVM
3. Bandingkan dengan Account Model Solana
