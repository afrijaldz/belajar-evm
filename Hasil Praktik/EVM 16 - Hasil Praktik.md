---
title: EVM 16 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 16 - Hasil Praktik

Hasil percobaan untuk materi [EVM 16 - Buyback-and-Burn Sendiri](../EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md). Dijalankan 2026-09-24 di fork Ethereum
(blok 26.046.487). Project: `~/Documents/riset/evm/buyback` — `src/GovToken.sol`, `src/BuybackBurner.sol`
(`BuybackBurner` + `FeeAuction`), `test/Buyback.t.sol`.

```bash
cd ~/Documents/riset/evm/buyback
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```

## Setup

- Pool Uniswap v2: 1.000.000 GOV + 100 ETH → 1 GOV = 0,0001 ETH.
- Fee protokol: 10 WETH.
- Desain B: `threshold` 90.000 GOV.

## Hasil (10 WETH fee)

| | A | B |
| --- | ---: | ---: |
| GOV terbakar | **90.661** | **90.000** (= threshold) |
| GOV per ETH fee | 9.066 | 9.000 |
| Nilai hilang | 9,3% — price impact + fee 0,3%, ditanggung protokol | **0,8%** margin searcher; impact ditanggung searcher (bayar 9,919 ETH untuk 90k GOV) |

## Desain A di bawah sandwich

Penyerang front-run 20 ETH sebelum transaksi keeper:

| `minGovOut` | Hasil |
| --- | --- |
| 0 | burn 63.956 GOV (**−29,45%**), penyerang **+3,03 ETH** |
| quote off-chain −0,5% | **revert** `INSUFFICIENT_OUTPUT_AMOUNT` |

## Desain B: persaingan dan kebocoran

- Dua searcher membaca `nonce = 0`; yang kedua **revert `InvalidNonce`**.
- Fee masuk 0,2 ETH per langkah. Searcher rasional melepas di langkah pertama yang untung setelah gas
  (0,005 ETH): wadah 10,0 ETH vs biaya 9,919 ETH → **bocor 80 bps** ke searcher.
- Buyback 10% pool membuang ~9% ke price impact di desain A. Satu `minOut = 0` = −29,5%.
