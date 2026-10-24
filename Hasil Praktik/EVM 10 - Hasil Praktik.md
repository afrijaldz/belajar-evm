---
title: EVM 10 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 10 - Hasil Praktik

Hasil percobaan untuk materi [EVM 10 - Burn _burn vs 0xdead](../EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md). Dijalankan 2026-09-24. Dua token
identik yang cuma beda cara burn, diuji dengan Foundry (4 test) dan diukur gas-nya di `anvil`. Ini
sisi "menulis" dari temuan riset tokenomics.

Project: `~/Documents/riset/evm/solidity-basics` — `src/BurnStyles.sol` (`SupplyBurnToken`,
`DeadBurnToken`, `SupplyReader`), `test/BurnStyles.t.sol`. Token latihan `MyToken` sengaja tidak
disentuh — latihan `burn` Sesi 8 masih untukmu.

## Satu burn, dua cerita

Bakar 300.000 dari supply 1.000.000: gaya A `totalSupply` 700.000; gaya B `totalSupply` 1.000.000
dan `balanceOf(0xdead)` 300.000. Supply efektif keduanya 700.000.

Akar kesalahan riset tokenomics: UNI tercatat "nol burn" padahal 112 jt dibakar; FDV CAKE terlihat
15,4x lebih besar.

## Supply efektif

Test `test_BurnSafeMustAlsoBeExcluded`: 20 sudah di `0xdead`, 30 antre di Safe burn. Cuma kurangi
`0xdead` → 80; kurangi `0xdead` + Safe burn → **50**.

## Gas (anvil, transaksi terpisah)

| | Gas |
| --- | ---: |
| `_burn()` (setiap kali) | **34.127** |
| → `0xdead` pertama | 51.935 (slot saldo `0xdead` baru, +17.100) |
| → `0xdead` berikutnya | 34.835 |

`_burn()` ~700 gas lebih murah. **Angka di dalam test Foundry justru terbalik** (18.224 vs 14.539,
bahkan dengan `vm.cool`) — test gas itu dihapus karena menyesatkan.

## Peta semua pola burn dari riset

| Token | Pola | `totalSupply` bisa dipercaya? | Cara membaca burn yang benar |
| --- | --- | --- | --- |
| **ETHFI** | A — `burn()` / `burnFrom()` | ✅ ya | `totalSupply()` atau `getPastTotalSupply()` (ETHFI Diperiksa dengan Saringan Lima Chain) |
| **UNI** | B — Firepit → `0xdead` (tidak ada `burn()`) | ❌ selalu 1 M | `balanceOf(0xdead)` ([EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) |
| **CAKE** | B + mint-lalu-bakar mingguan | ❌ **naik terus** (2,44 M dalam 10 bulan) | `totalSupply − 0xdead − Safe burn − MasterChefV2` (CAKE Diperiksa Ulang 2026-09-24) |
| **BNB** | coin native → `0xdead` di BSC | tidak ada `totalSupply()` | `eth_getBalance(0xdead)` (BNB Diperiksa Ulang 2026-09-24) |
| **HYPE** | native HyperCore/HyperEVM, supply berkurang langsung | ✅ dari API `tokenDetails` | `1 M − totalSupply`; alamat burn cuma 1.676 (HYPE Diperiksa Ulang 2026-09-24) |

Kesalahan di riset (UNI "nol burn", HYPE "1.676 dibakar") terjadi karena memakai metode gaya A untuk
token gaya B, dan sebaliknya.
