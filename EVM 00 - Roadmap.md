---
title: EVM 00 - Roadmap
tags: [learning, evm, ethereum, bsc, solidity]
updated: 2026-09-24
---

# EVM Roadmap

Tujuan: naik level dari **pembaca data on-chain** ke **smart contract developer di EVM**. Kamu
sudah bisa membaca state EVM lewat RPC (`eth_call`, `eth_getBalance`, selector, `cast`, node
arsip) dari riset tokenomics 2026-09-24. Sekarang fokus ke sisi kontrak: Solidity + Foundry.

Level awal: beginner (kontrak), intermediate (membaca on-chain).

Project praktik ada di `~/Documents/riset/evm/` — lihat kolom Kode di tabel urutan baca.

Konvensi nama catatan sesi baru: `EVM NN - Judul.md` (dua digit).

Satu chain untuk praktik: **Ethereum mainnet fork + BSC fork** lewat `anvil`. Mesinnya sama
(lihat [EVM 01 - EVM vs Non-EVM](EVM%2001%20-%20EVM%20vs%20Non-EVM.md)), jadi semua materi berlaku di keduanya.

## Status modul (2026-10-03)

**0 dari 24 sesi selesai.** Catatan dan kode sesi 1–24 sudah ada (dibuat 2026-09-24), tapi pada 2026-10-03 semua sesi ditandai belum selesai atas permintaan user: dipelajari ulang satu per satu. Yang juga tersisa:

| Sisa | Butuh |
| --- | --- |
| Deploy + verifikasi sungguhan (Sesi 21), deploy `SupplyWatch` (Sesi 24) | saldo faucet ≥0,005 ETH ke `0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56` (EVM Testnet Deployer) |
| Latihan `DepositBank` (Sesi 7) | dikerjakan sendiri: `solidity-basics/src/Exercise.sol` |
| Latihan `burn` (Sesi 8) | dikerjakan sendiri: `solidity-basics/test/ExerciseBurn.t.sol` |

## Cara membaca

Baca catatan **berurutan dari nomor terkecil**. Semua catatan sesi di folder ini diberi awalan
`EVM NN - …`, jadi di explorer Obsidian sudah tersusun sesuai urutan. Modul ini punya folder sendiri, `Learning/EVM/`. Riset tokenomics yang dipakai sebagai
bahan praktik (UNI, ETHFI, CAKE, BNB, …) ada di folder chain masing-masing — lihat bagian
"Bahan praktik dari riset".

Rangkuman dengan kata-kata sendiri ditulis di Rangkuman EVM (satu section per sesi; Claude mengoreksi hanya kalau diminta).

Tiap catatan berdiri sendiri, tapi sering menaut ke nomor sebelumnya — kalau ada istilah yang
asing, ikuti link-nya mundur.

## Urutan baca

| No | Catatan | Isi | Kode | Latihan |
| ---: | --- | --- | --- | --- |
| 00 | (catatan ini) | peta belajar, progress, environment | — | — |
| **Fase 1** | **Dasar EVM** | | | |
| 01 | [EVM 01 - EVM vs Non-EVM](EVM%2001%20-%20EVM%20vs%20Non-EVM.md) | mesin vs native token, BEP-20 = ERC-20, HyperCore vs HyperEVM; detail per komponen (opcode, stack, memory, storage, PC, gas, alamat, RPC, native token, EVM vs node vs validator) di folder `EVM 01 - Detail/` (`EVM 01.01`–`01.13`) | — | — |
| 02 | [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md) | empat field akun, alamat kontrak, storage mentah, `0xdead`, EIP-7702; detail di folder `EVM 02 - Detail/` (`EVM 02.01`–`02.02`) | `account-model` | — |
| 03 | [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md) | anatomi transaksi, EIP-1559, rumus base fee, harga calldata/storage, log | `account-model` | — |
| 04 | [EVM 04 - ABI](EVM%2004%20-%20ABI.md) | selector, encoding, event, decode transaksi burn CAKE | `account-model` | — |
| 05 | [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md) | fork BSC, archive node, impersonate, `setStorageAt`, `forge test` di fork | `account-model` | — |
| 06 | [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md) | bot searcher, Firepit + TokenJar, burn UNI sendiri, aturan mint 2% | — | — |
| **Fase 2** | **Solidity dan Token** | | | |
| 07 | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md) | overflow, packing, storage/memory/calldata, visibility, modifier, custom error, `payable`/`receive()`; detail di folder `EVM 07 - Detail/` (`EVM 07.01`–`07.02`) | `solidity-basics` | ☐ `DepositBank` |
| 08 | [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md) | token dari nol, fuzz test, bug self-transfer | `solidity-basics` | ☐ `burn` |
| 09 | [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) | race approve, unlimited approve, phishing permit, USDT | `solidity-basics` | — |
| 10 | [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md) | dua gaya burn, supply efektif, peta pola burn riset | `solidity-basics` | — |
| 11 | [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) | indexer mini, filter topic, batas `eth_getLogs`, reorg | `indexer` | — |
| 12 | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) | vault ETH + ERC-20, 3 jebakan, invariant + mutation test | `vault-project` | — |
| **Fase 3** | **DeFi dan Integrasi** | | | |
| 13 | [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) | harga dari reserve, rumus swap, price impact, fee protokol → TokenJar | `amm` | — |
| 14 | [EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md) | deadline, `amountOutMin`, simulasi sandwich, quote on-chain bukan proteksi | `swap` | — |
| 15 | [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md) | `sqrtPriceX96` dan tick, v3 vs v2, posisi sempit/lebar/penuh, keluar rentang | `uniswap-v3` | — |
| 16 | [EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md) | desain keeper (CAKE) vs lelang (UNI), sandwich, kebocoran, threshold | `buyback` | — |
| 17 | [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md) | `delegatecall`, tabrakan storage, EIP-1967, upgrade append-only, initializer, UUPS, Safe dan USDC; detail di folder `EVM 17 - Detail/` (`EVM 17.01`) | `proxy` | — |
| 18 | [EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md) | oracle spot vs TWAP + flash loan Morpho, `tx.origin`, **checklist auditor** | `security` | — |
| **Fase 4** | **Production** | | | |
| 19 | [EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md) | coverage cabang, differential vs router asli, ghost variable, shrinking, fork cache | `vault-project`, `swap` | — |
| 20 | [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md) | baseline + benchmark transaksi nyata, optimizer, `encodeCall`, `unchecked`, trade-off storage | `vault-project` | — |
| 21 | [EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md) | wallet deployer, RPC testnet, `forge script`, simulasi + fork, prinsip verifikasi | `vault-project` | ☐ deploy sungguhan (butuh faucet) |
| 22 | [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md) | monitor event + solvency + reorg, laju burn UNI, burn L2 lewat bridge | `monitor` | — |
| 23 | [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md) | 4 chain berdampingan, `block.number` Arbitrum, CREATE vs CREATE2, replay, invariant bridge UNI | `multichain` | — |
| 24 | [EVM 24 - Capstone SupplyWatch](EVM%2024%20-%20Capstone%20SupplyWatch.md) | kontrak supply efektif on-chain tanpa owner, fork UNI + CAKE, CREATE2 multi-chain | `capstone` | ☐ deploy sungguhan (bersama Sesi 21) |

Folder kode ada di `~/Documents/riset/evm/<nama>`. Latihan ☐ = belum dikerjakan; cara mengecek ada di
catatan sesinya.

## Fase 1: Dasar EVM (Sesi 1-6)

- [ ] Sesi 1: EVM vs non-EVM: mesin eksekusi vs native token, chain ID, BEP-20 = ERC-20
- [ ] Sesi 2: Model akun: EOA vs contract account, nonce, balance, code, storage
- [ ] Sesi 3: Transaksi dan gas: gas limit, base fee, priority fee, EIP-1559, receipt, log
- [ ] Sesi 4: ABI: function selector, encoding argumen, event topic (bongkar ulang `0x70a08231` dan `Transfer`)
- [ ] Sesi 5: Setup Foundry + `anvil` lokal + fork mainnet/BSC
- [ ] Sesi 6: Baca kontrak nyata di fork: CAKE, UNI, MasterChef — dengan `cast` dan `forge` (CAKE + MasterChef di Sesi 5, UNI di Sesi 6)

## Fase 2: Solidity dan Token (Sesi 7-12)

- [ ] Sesi 7: Dasar Solidity: tipe data, storage vs memory, visibility, modifier (latihan `DepositBank` belum dikerjakan)
- [ ] Sesi 8: Tulis ERC-20 sendiri + test di Foundry (latihan `burn` belum dikerjakan)
- [ ] Sesi 9: `approve` / `transferFrom` / allowance, dan kenapa approval bisa berbahaya
- [ ] Sesi 10: Burn: `_burn()` vs kirim ke `0xdead` — praktikkan dua-duanya, bandingkan `totalSupply`
- [ ] Sesi 11: Events dan indexing: `eth_getLogs`, topic filter, batas blok RPC
- [ ] Sesi 12: Mini project: vault deposit/withdraw ERC-20 + ETH

## Fase 3: DeFi dan Integrasi (Sesi 13-18)

- [ ] Sesi 13: AMM `x·y=k`: baca pool Uniswap v2 / PancakeSwap v2 di fork
- [ ] Sesi 14: Swap dari kontrak: router, slippage, deadline
- [ ] Sesi 15: Konsentrasi likuiditas: tick dan posisi di Uniswap v3 (bandingkan Raydium CLMM)
- [ ] Sesi 16: Buyback-and-burn dari kontrak: tulis versi sendiri (sisi baca CAKE sudah di [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md), UNI Firepit/TokenJar di [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md))
- [ ] Sesi 17: Proxy dan upgrade: Gnosis Safe, transparent/UUPS proxy, `masterCopy()`
- [ ] Sesi 18: Security basics: reentrancy, access control, oracle manipulation, audit mindset (reentrancy sudah dipraktikkan di Sesi 12)

## Fase 4: Production (Sesi 19-24)

- [ ] Sesi 19: Testing lanjutan: fuzz, invariant, fork test di Foundry (dasar fuzz/invariant/fork sudah di Sesi 5, 8, 12)
- [ ] Sesi 20: Gas optimization: storage slot, packing, `calldata`
- [ ] Sesi 21: Deploy ke testnet, verifikasi source di explorer (persiapan selesai; deploy menunggu saldo faucet ke `0x54f4…db56`)
- [ ] Sesi 22: Monitoring: event listener, alert saldo/transfer
- [ ] Sesi 23: Multi-chain: deploy kontrak yang sama ke BSC, Base, Arbitrum (di fork; deploy sungguhan menunggu faucet Sesi 21)
- [ ] Sesi 24: Capstone: kontrak kecil yang memantau burn/buyback token dan bisa dicek siapa pun

## Progress Log

| Sesi | Tanggal | Materi | Catatan |
| --- | --- | --- | --- |
| 1 | 2026-09-24 | EVM vs non-EVM | Muncul dari riset tokenomics BSC/Ethereum; ditulis di [EVM 01 - EVM vs Non-EVM](EVM%2001%20-%20EVM%20vs%20Non-EVM.md) |
| 2 | 2026-09-24 | EOA vs contract, nonce, code, storage, EIP-7702 | Praktik di `anvil` + cek akun nyata BSC; ditulis di [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md). Bonus: delegasi EIP-7702 |
| 3 | 2026-09-24 | Transaksi, EIP-1559, calldata, SSTORE, revert, log | Rumus base fee terbukti sampai 1 wei; calldata 10/40 gas (EIP-7623); ditulis di [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md) |
| 4 | 2026-09-24 | Selector, encoding statis/dinamis, event, decode tx CAKE | Pipeline Safe → MultiSend → Timelock → `burnCake` mingguan terbongkar; ditulis di [EVM 04 - ABI](EVM%2004%20-%20ABI.md) |
| 5 | 2026-09-24 | Fork BSC, impersonate, setStorageAt, forge test fork | Burn mingguan CAKE direplay identik; syarat archive node; aliasing sampel MCv2; ditulis di [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md) |
| 6 | 2026-09-24 | Burn UNI: bot searcher, Firepit + TokenJar, burn sendiri di fork, aturan mint 2% | TokenJar kosong saat istirahat; checkpoint suara UNI merusak storage hack; mint 2% terbukti; ditulis di [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md) |
| 7 | 2026-09-24 | Tipe & overflow, packing, storage/memory/calldata, visibility, modifier, custom error | 18 test; packing hemat 21.850 gas; `private` terbaca dari slot; latihan 3 task; ditulis di [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md) |
| 8 | 2026-09-24 | ERC-20 dari nol, fuzz test, bug self-transfer, deploy anvil | Selector/event identik dengan CAKE/UNI; selisih gas holder baru 17.100; ditulis di [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md) |
| 9 | 2026-09-24 | Race approve, unlimited approve, permit phishing, revoke, USDT | 7 test serangan; permit menguras tanpa transaksi korban; USDT zero-first + tanpa bool; ditulis di [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) |
| 10 | 2026-09-24 | `_burn()` vs `0xdead`, pembaca supply efektif, peta pola burn riset | Gas nyata setara (34.127 vs 34.835); angka gas di test terbalik; ditulis di [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md) |
| 11 | 2026-09-24 | Indexer mini, filter topic, batas getLogs 4 RPC, fetch adaptif, reorg, biaya event | Saldo 6 holder direbuild tepat; alamat nyasar 11,3% supply ditemukan; event 40–50x lebih murah; ditulis di [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) |
| 12 | 2026-09-24 | Vault ETH + ERC-20, 3 jebakan (reentrancy, fee token, USDT), invariant + mutation test | Fase 2 selesai; bug handler 20% revert ditemukan; ditulis di [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) |
| 13 | 2026-09-24 | Harga dari reserve, rumus swap, price impact, k naik, fee protokol v2, Pancake 0,25% | Rumus cocok sampai sen; feeTo Uniswap v2 = TokenJar; feeTo Pancake v2 = sumber buyback CAKE; ditulis di [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) |
| 14 | 2026-09-24 | Router, deadline, amountOutMin, sandwich 3 skenario | Tanpa proteksi korban −30,8%, slippage 0,5% membatasi ke −0,40%, quote on-chain = tanpa proteksi; ditulis di [EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md) |
| 15 | 2026-09-24 | Uniswap v3: harga dari sqrtPriceX96, v3 vs v2, 3 posisi, fee, keluar rentang | L sempit ~400x, fee 418x; keluar rentang → 100% WETH; jebakan gas 63/64; ditulis di [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md) |
| 16 | 2026-09-24 | Buyback keeper vs lelang harga tetap, GOV + pool v2 di fork | Efisiensi ~90% keduanya (kedalaman pool); sandwich −29,5% di desain A; bocor 0,8% di desain B; ditulis di [EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md) |
| 17 | 2026-09-24 | Proxy sendiri, 3 bug upgrade, UUPS beku, bedah Safe dan USDC | Safe slot 0 disengaja; USDC admin = EOA di slot ZeppelinOS; UNI/ETHFI/CAKE bukan proxy; ditulis di [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md) |
| 18 | 2026-09-24 | Oracle manipulation dengan flash loan, TWAP, tx.origin, checklist auditor | Spot: lender 0/200, penyerang +199 ETH; TWAP: lender 198/200; biaya manipulasi ~0,55 ETH; Fase 3 selesai; ditulis di [EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md) |
| 19 | 2026-09-24 | Coverage, differential, ghost + shrinking, rpc_endpoints + cache | Cabang Vault 40% → 100%, pengecekan mati ditemukan; fuzz membongkar rumus 'fee duluan'; shrink 17 → 2; ditulis di [EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md) |
| 20 | 2026-09-24 | Optimasi Vault satu langkah per ukuran, benchmark anvil | Optimizer deploy −44%; encodeCall 0 gas; hapus 1 slot −33% tapi dipertahankan demi audit; ditulis di [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md) |
| 21 | 2026-09-24 | Wallet deployer, script deploy, simulasi Sepolia + fork, prinsip verifikasi | 8 tx / 1,19 jt gas / ~0,0013 ETH; bytecode identik termasuk metadata; menunggu faucet; ditulis di [EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md) |
| 22 | 2026-09-24 | Monitor Vault (event, alert, solvency, reorg), laju burn UNI mainnet | Eksploit + reorg terdeteksi; burn UNI ~96% dari Firepit L2 lewat bridge, bukan mainnet; ditulis di [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md) |
| 23 | 2026-09-24 | Multi-chain: CREATE2 alamat sama, block.number Arbitrum = L1, replay, bridge UNI | CREATE2 sama di Base/BSC fork; escrow ≥ supply di 3 L2, ~128 rb UNI burn dalam perjalanan; ditulis di [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md) |
| 24 | 2026-09-24 | Capstone SupplyWatch: on-chain, tanpa owner, checkpoint 1 slot, CREATE2 | UNI −3,73%/th (30 hari); CAKE −8,25%/th sama persis dengan riset; alamat sama di 3 chain; ditulis di [EVM 24 - Capstone SupplyWatch](EVM%2024%20-%20Capstone%20SupplyWatch.md) |

## Bahan praktik dari riset

Riset 2026-09-24 sudah memakai banyak teknik EVM secara langsung. Pakai sebagai contoh nyata
saat sesi terkait:

- UNI Diperiksa dengan Saringan Lima Chain — token tanpa fungsi `burn`, burn ke `0xdead`
  (Sesi 10), riwayat transfer via Blockscout.
- ETHFI Diperiksa dengan Saringan Lima Chain — ERC20Votes, `getPastTotalSupply()` untuk
  riwayat tanpa log (Sesi 11).
- CAKE Diperiksa Ulang 2026-09-24 — node arsip, `eth_getLogs` per 40k blok, binary search
  blok, MasterChef mint-and-burn (Sesi 6, 11, 16).
- BNB Diperiksa Ulang 2026-09-24 — native coin tanpa `totalSupply()`, `eth_getBalance`
  historis (Sesi 2).
- Ke Mana Emisi CAKE Mengalir — identifikasi proxy Gnosis Safe dari bytecode (Sesi 17).

## Referensi

- Ethereum docs (EVM, akun, gas): https://ethereum.org/en/developers/docs/
- Solidity docs: https://docs.soliditylang.org/
- Foundry Book: https://book.getfoundry.sh/
- evm.codes (opcode dan gas): https://www.evm.codes/
- OpenZeppelin Contracts: https://docs.openzeppelin.com/contracts/
- BNB Chain docs: https://docs.bnbchain.org/

## Environment (2026-09-24)

- Foundry 1.7.1 sudah terpasang: `forge`, `cast`, `anvil`.
- `solc` tidak terpasang terpisah — Foundry mengunduh compiler sendiri saat `forge build`.
- Node v24.18.0 (nvm), untuk script viem/ethers kalau perlu.
- RPC yang terbukti jalan:
  - Ethereum: `https://ethereum-rpc.publicnode.com` (bukan arsip; `eth_getLogs` historis ditolak)
  - BSC arsip: `https://bsc-mainnet.nodereal.io/v1/<NODEREAL_API_KEY>` (rate limit;
    untuk fork pakai `--compute-units-per-second 20`). RPC BSC gratis lain gagal untuk fork —
    lihat [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)
