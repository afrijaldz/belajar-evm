---
title: EVM 13 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 13 - Hasil Praktik

Hasil percobaan untuk materi [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md). Dijalankan 2026-09-24 — sesi pertama Fase 3.
Pool Uniswap v2 dibaca dan di-swap di fork Ethereum (blok ~26.046.376, `eth.drpc.org`), PancakeSwap
v2 dibaca langsung di BSC.

Script: `~/Documents/riset/evm/amm/` — `swaps.py` dan `feeto.py` butuh `anvil --fork-url` Ethereum
di port 8545; `pancake.py` langsung ke nodereal BSC.

## Pair Uniswap v2 WETH/USDC

`0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc`:

| | |
| --- | --- |
| Reserve | 10.441.265 USDC + 3.918,52 WETH (TVL ≈ $20,9 jt) |
| **Harga spot** | 10.441.265 / 3.918,52 = **$2.664,59 / ETH** |
| `token0` / `token1` | USDC / WETH (diurutkan berdasarkan alamat) |

## Rumus vs router vs swap nyata

| ETH masuk | % reserve | Rumus sendiri | Router | Swap nyata | Harga efektif | Price impact |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 0,03% | 2.655,92 | 2.655,92 | 2.655,92 | 2.655,92 | 0,33% |
| 10 | 0,26% | 26.498,55 | 26.498,55 | 26.498,55 | 2.649,85 | 0,55% |
| 100 | 2,55% | 259.068,16 | 259.068,16 | 259.068,16 | 2.590,68 | 2,77% |
| 1.000 | 25,52% | 2.117.768,01 | 2.117.768,01 | 2.117.768,01 | **2.117,77** | **20,52%** |

**Cocok sampai sen di keempat ukuran.** Tiap swap dimulai dari state sama (`evm_snapshot`/`evm_revert`).
Swap kecil: impact ≈ fee 0,3%. Swap 25% reserve: **−20,5%**. Ini alasan saringan riset memakai
**likuiditas relatif terhadap mcap**.

## k naik

`k` naik 0,0001% setelah swap 1 ETH dan 0,061% setelah swap 1.000 ETH.

## Fee protokol Uniswap v2

Fee switch ≈17% — angka "17% fee v2" di UNI Diperiksa dengan Saringan Lima Chain.

| | |
| --- | --- |
| `factory.feeTo()` | **`0xf38521f130fcCF29dB1961597bc5d2B60F995f85` = TokenJar** ([EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) |
| Pending sebelum aksi | ~0 — bot baru saja memicu `mint` |
| Setelah swap 0,1 ETH + tambah likuiditas | `Mint LP 0x0 → TokenJar` **461.853.107** unit |

Ini menjelaskan transaksi bot di Sesi 6 yang berisi log `Mint` lalu `Burn` di puluhan pair.

## PancakeSwap v2 (BSC)

Pair CAKE/WBNB `0x0eD7e52944161450477ee417DE9Cd3a859b14fD0`: 4.182.593 CAKE + 14.329,24 WBNB.

| BNB masuk | Rumus 9975/10000 | Rumus 997/1000 | Router Pancake |
| ---: | ---: | ---: | ---: |
| 1 | 291,1422 | 290,9963 | **291,1422** ✓ |
| 100 | 28.914,9638 | 28.900,5702 | **28.914,9638** ✓ |

- **Fee 0,25% terbukti** dari kecocokan rumus.
- Harga dari reserve: 1 CAKE = 0,003426 BNB × $772,78 = **$2,65** (CoinGecko $2,63).
- **`factory.feeTo()` = `0x0ED943Ce24BaEBf257488771759F9BF482C39706`** — salah satu dari tiga sumber
  buyback yang mengirim CAKE ke Safe burn `0xceba…` di CAKE Diperiksa Ulang 2026-09-24
  (~208.757 CAKE/minggu). Di riset alamat ini masih tanpa nama; sekarang teridentifikasi sebagai
  penerima fee protokol Pancake v2.
