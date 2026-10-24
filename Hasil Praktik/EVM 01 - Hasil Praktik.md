---
title: EVM 01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md). Dijalankan 2026-09-24 di Ethereum
mainnet dan BSC mainnet dengan `cast` (Foundry).

## Perintah yang sama di chain berbeda

Membaca token di BSC (CAKE Diperiksa Ulang 2026-09-24, BNB Diperiksa Ulang 2026-09-24)
memakai perintah yang persis sama dengan membaca token di Ethereum
(UNI Diperiksa dengan Saringan Lima Chain, ETHFI Diperiksa dengan Saringan Lima Chain):

- `eth_call` dengan selector `0x70a08231` (`balanceOf`) dan `0x18160ddd` (`totalSupply`)
- `eth_getBalance` untuk membaca BNB di `0xdead`
- `cast` pindah chain cukup dengan mengganti `--rpc-url`
- alamat `0x000000000000000000000000000000000000dEaD` berlaku di kedua chain

## Block time BSC

Block time BSC terukur **0,45 detik** (2026-09-24), dibandingkan 12 detik di Ethereum.
