---
title: EVM 15 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 15 - Hasil Praktik

Hasil percobaan untuk materi [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](../EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md). Dijalankan
2026-09-24. Pool v3 WETH/USDC 0,05% dibaca di fork Ethereum (blok ~26.046.443, `eth.drpc.org`),
dibandingkan dengan v2 dari [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md), lalu tiga posisi dibuat sendiri.

Script: `~/Documents/riset/evm/uniswap-v3/` — `read_pool.py`, `compare_depth.py`, `positions.py`,
lalu `stress.py`. Semua butuh `anvil --fork-url https://eth.drpc.org` di port 8545.

## Membaca pool

Pool `0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640` (token0 USDC, token1 WETH, fee 500 = 0,05%,
`tickSpacing` 10).

| Field | Nilai | Arti |
| --- | --- | --- |
| `slot0.sqrtPriceX96` | 1,540 × 10³³ | `√harga × 2⁹⁶` (fixed-point Q64.96) |
| `slot0.tick` | 197.510 | `floor(log₁.₀₀₀₁(harga))` — dihitung ulang, cocok |
| `liquidity` | 2,87 × 10¹⁸ | `L` yang aktif di rentang tick sekarang saja |

Harga USDC/ETH = **$2.646,47**. Rentang aktif: tick [197.510, 197.520) = **$2.643,90–$2.646,54**
(lebar 0,1%).

## v3 vs v2: kedalaman

Quote `QuoterV2.quoteExactInputSingle` (v3) vs `getAmountsOut` (v2), blok sama (price impact):

| ETH → USDC | v3 0,05% | v2 0,3% |
| ---: | ---: | ---: |
| 1 | 0,05% | 0,08% |
| 100 | **0,23%** | 2,53% |
| 1.000 | **1,84%** (36 tick terlewati) | **20,29%** |
| 5.000 | quote gagal (terlalu banyak tick untuk satu `eth_call`) | — |

TVL v3 ≈ **$99 jt** (74,2 jt USDC + 9.362 WETH), v2 ≈ $20,9 jt → 4,7x. Tapi impact 1.000 ETH **11x
lebih kecil**.

## Tiga posisi, modal sama (~$53 rb terpakai masing-masing)

| Posisi | Rentang | NFT | `L` per $ | vs penuh |
| --- | --- | --- | ---: | ---: |
| **Sempit ±0,5%** | $2.625–$2.652 | #1371675 | 3,90 × 10¹² | **~400x** |
| Lebar ±50% | $1.320–$5.282 | #1371676 | 3,32 × 10¹⁰ | 3,4x |
| Penuh (= v2) | 0–∞ | #1371677 | 9,73 × 10⁹ | 1x |

### Fee

5 kali bolak-balik 20 WETH (~$527 rb volume), harga tetap di dalam rentang sempit.

| Posisi | Fee (`collect` statis) | vs penuh |
| --- | ---: | ---: |
| **Sempit** | **$20,89** | **418x** |
| Lebar | $0,18 | 3,6x |
| Penuh | $0,05 | 1x |

Rasio fee ≈ rasio `L` (400x).

### Harga keluar rentang

Jual 600 WETH → harga $2.637 → **$2.583** (−2,0%), di bawah batas bawah posisi sempit.

| Posisi | Isi (`decreaseLiquidity` statis) |
| --- | --- |
| **Sempit** | **0 USDC + 20,36 WETH — 100% WETH** |
| Lebar | 25.431 USDC + 10,38 WETH (51% WETH) |
| Penuh | 26.105 USDC + 10,11 WETH (50% WETH) |

## Jebakan yang ditemui

- Swap lewat router revert `OutOfGas` di 154.175 dari limit hasil estimasi 156.106 (aturan 63/64,
  EIP-150).
- `cast send` tanpa `--json` tidak selalu gagal saat transaksi revert.
- `cast keccak` menolak `--rpc-url` (perintah lokal).
- State fork menumpuk antar percobaan: tiap run `positions.py` yang gagal tetap menjual 30 WETH,
  harga bergeser $2.646 → $2.638.
