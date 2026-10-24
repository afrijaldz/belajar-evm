---
title: EVM 14 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 14 - Hasil Praktik

Hasil percobaan untuk materi [EVM 14 - Swap dari Kontrak dan Sandwich](../EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md). Dijalankan 2026-09-24.
Kontrak `SwapHelper` melakukan swap lewat router Uniswap v2; serangan sandwich disimulasikan di fork
Ethereum (blok 26.046.409, `eth.drpc.org`) dengan `forge test`.

Project: `~/Documents/riset/evm/swap` — `src/SwapHelper.sol`, `test/Sandwich.t.sol`.

```bash
cd ~/Documents/riset/evm/swap
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```

## Dua pengaman di router

`deadline` dan `amountOutMin` terbukti di test `test_ExpiredDeadlineReverts` dan
`test_MinOutAboveQuoteReverts`.

## Sandwich

Korban: swap **50 ETH → USDC** di pool WETH/USDC (~3.918 WETH). Tanpa serangan: **131.058 USDC**.

### Tanpa proteksi (`amountOutMin = 0`)

| Front-run | Korban dapat | Kerugian korban | Untung penyerang |
| ---: | ---: | ---: | ---: |
| 1 ETH | 131.002 | 0,05% | 0,019 ETH |
| 10 ETH | 130.407 | 0,50% | 0,194 ETH |
| 50 ETH | 127.812 | 2,48% | 0,954 ETH |
| 200 ETH | 118.741 | 9,40% | 3,589 ETH |
| **800 ETH** | **90.641** | **30,84%** (−40.417 USDC) | **11,498 ETH (~$30.600)** |

### Slippage 0,5% dihitung off-chain

`amountOutMin` = 130.413 (quote × 0,995).

| Front-run | Hasil |
| ---: | --- |
| 1 / 3 / 5 / **8** ETH | korban lolos, kerugian 0,05% / 0,15% / 0,25% / **0,40%**; penyerang +0,019 … **+0,155 ETH** |
| 10 ETH ke atas | **transaksi korban revert** — penyerang rugi fee (−0,059 ETH di 10 ETH) |

Dengan 0,5%, penyerang mengambil hampir sampai batas (0,40%).

### Quote di dalam transaksi

`swapWithOnchainQuote(50 bps)` menghasilkan angka **identik** dengan tanpa proteksi di semua ukuran
(sampai −30,84%).

## Catatan

- Script Sesi 13 ([EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md)) memakai `amountOutMin = 0` — aman hanya di fork.
- Jebakan Foundry yang ditemui: versi pertama test menghitung profit dari `attacker.balance` setelah
  `vm.prank(attacker)` + `call{value: x}`, dan hasilnya mengikutkan modal (front-run 10 ETH →
  "profit" 10,194 ETH). Perbaikan: profit = ETH hasil jual − ETH front-run, diambil dari return value
  swap.
