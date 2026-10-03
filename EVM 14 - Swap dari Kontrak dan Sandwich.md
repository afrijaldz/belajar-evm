---
title: EVM 14 - Swap dari Kontrak dan Sandwich
tags: [learning, evm, defi, uniswap, mev, slippage]
updated: 2026-09-24
---

# Swap dari Kontrak dan Sandwich

Catatan dari Sesi 14 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Kontrak `SwapHelper` melakukan swap lewat
router Uniswap v2; serangan sandwich disimulasikan di fork Ethereum (blok 26.046.409, `eth.drpc.org`)
dengan `forge test`. Lanjutan dari [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md).

Project: `~/Documents/riset/evm/swap` — `src/SwapHelper.sol`, `test/Sandwich.t.sol`.

```bash
cd ~/Documents/riset/evm/swap
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```

## Dua pengaman di router

| Parameter | Revert kalau | Melindungi dari |
| --- | --- | --- |
| `deadline` | `block.timestamp > deadline` → `UniswapV2Router: EXPIRED` | transaksi yang ditahan di mempool / oleh validator sampai harga berubah |
| `amountOutMin` | hasil < min → `UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT` | harga bergerak sebelum transaksi dieksekusi, termasuk sandwich |

Keduanya terbukti di test (`test_ExpiredDeadlineReverts`, `test_MinOutAboveQuoteReverts`).

## Sandwich

Urutan dalam satu blok: **penyerang beli → korban beli → penyerang jual**. Pembelian penyerang
menaikkan harga, korban membeli di harga lebih buruk dan mendorong harga lebih jauh, penyerang
menjual di harga tertinggi itu.

Korban: swap **50 ETH → USDC** di pool WETH/USDC (~3.918 WETH). Tanpa serangan: **131.058 USDC**.

### Tanpa proteksi (`amountOutMin = 0`)

| Front-run | Korban dapat | Kerugian korban | Untung penyerang |
| ---: | ---: | ---: | ---: |
| 1 ETH | 131.002 | 0,05% | 0,019 ETH |
| 10 ETH | 130.407 | 0,50% | 0,194 ETH |
| 50 ETH | 127.812 | 2,48% | 0,954 ETH |
| 200 ETH | 118.741 | 9,40% | 3,589 ETH |
| **800 ETH** | **90.641** | **30,84%** (−40.417 USDC) | **11,498 ETH (~$30.600)** |

Kerugian tidak terbatas: makin besar modal penyerang, makin banyak yang diambil.

### Slippage 0,5% dihitung off-chain (sebelum kirim)

`amountOutMin` = 130.413 (quote × 0,995).

| Front-run | Hasil |
| ---: | --- |
| 1 / 3 / 5 / **8** ETH | korban lolos, kerugian 0,05% / 0,15% / 0,25% / **0,40%**; penyerang +0,019 … **+0,155 ETH** |
| 10 ETH ke atas | **transaksi korban revert** — penyerang rugi fee (−0,059 ETH di 10 ETH) |

- **Slippage tolerance = batas maksimal yang rela diserahkan ke penyerang.** Dengan 0,5%, penyerang
  mengambil hampir sampai batas (0,40%).
- Di dunia nyata penyerang mengirim front-run + back-run sebagai **bundle atomik**: kalau korban revert,
  bundle tidak dieksekusi — penyerang tidak rugi. Korban cuma kehilangan gas.

### "Proteksi" palsu: quote di dalam transaksi

`swapWithOnchainQuote(50 bps)` menghitung `amountOutMin` dari `getAmountsOut` **di dalam** transaksi.
Hasilnya **identik** dengan tanpa proteksi di semua ukuran (sampai −30,84%): quote diambil **setelah**
front-run menggeser harga. `amountOutMin` harus dihitung off-chain sebelum mengirim, seperti wallet.

## Pelajaran

1. Jangan pernah `amountOutMin = 0` di mainnet (script Sesi 13 memakainya — aman hanya di fork).
2. Slippage sekecil yang masih wajar untuk ukuran order dan likuiditas pool.
3. Quote on-chain dari pool yang sama bukan pengaman.
4. Deadline pendek; jangan `type(uint256).max`.
5. Order besar di pool dangkal: kirim lewat mempool privat (mis. Flashbots Protect) atau pecah.

## Jebakan Foundry

`vm.prank(attacker)` + `call{value: x}` **tidak memotong ETH dari `attacker`** — `msg.sender` berganti,
tapi saldo attacker tidak turun. Versi pertama test menghitung profit dari `attacker.balance` dan
hasilnya mengikutkan modal (front-run 10 ETH → "profit" 10,194 ETH). Perbaikan: profit = ETH hasil
jual − ETH front-run, diambil dari return value swap.
