---
title: EVM 16 - Buyback-and-Burn Sendiri
tags: [learning, evm, defi, tokenomics, buyback, burn]
updated: 2026-09-24
---

# Buyback-and-Burn Sendiri

Catatan dari Sesi 16 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Dua desain buyback-and-burn ditulis dan diuji di
fork Ethereum (blok 26.046.487) dengan token `GOV` sendiri + pool Uniswap v2 asli. Desainnya meniru dua
mesin yang sudah dibongkar: pipeline CAKE ([EVM 04 - ABI](EVM%2004%20-%20ABI.md), [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)) dan Firepit UNI
([EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)). Lanjutan dari [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md).

Project: `~/Documents/riset/evm/buyback` — `src/GovToken.sol`, `src/BuybackBurner.sol`
(`BuybackBurner` + `FeeAuction`), `test/Buyback.t.sol`.

```bash
cd ~/Documents/riset/evm/buyback
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```

## Setup

- `GovToken`: `burn()`/`burnFrom()` **mengurangi `totalSupply`** (gaya A, [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md)) —
  supply selalu jujur, tidak perlu rumus supply efektif.
- Pool Uniswap v2: **1.000.000 GOV + 100 ETH** → 1 GOV = 0,0001 ETH.
- Fee protokol: 10 WETH.

## Dua desain

| | A — `BuybackBurner` (gaya CAKE) | B — `FeeAuction` (gaya UNI Firepit) |
| --- | --- | --- |
| Siapa beli | keeper terpercaya | siapa saja (searcher) |
| Cara | swap WETH → GOV di AMM, `burn` hasilnya | bayar `threshold` GOV (dibakar), ambil seluruh isi wadah |
| Butuh AMM/oracle di kontrak | ya | tidak |
| Pengaman | `minGovOut` dari off-chain + `deadline` | `nonce` (hanya satu pemanggil menang) |

## Hasil (10 WETH fee)

| | A | B |
| --- | ---: | ---: |
| GOV terbakar | **90.661** | **90.000** (= threshold) |
| GOV per ETH fee | 9.066 | 9.000 |
| Nilai hilang | 9,3% — price impact + fee 0,3%, ditanggung protokol | **0,8%** margin searcher; impact ditanggung searcher (bayar 9,919 ETH untuk 90k GOV) |

### Desain A di bawah sandwich

Penyerang front-run 20 ETH sebelum transaksi keeper:

| `minGovOut` | Hasil |
| --- | --- |
| 0 | burn 63.956 GOV (**−29,45%**), penyerang **+3,03 ETH** |
| quote off-chain −0,5% | **revert** `INSUFFICIENT_OUTPUT_AMOUNT` |

### Desain B: persaingan dan kebocoran

- Dua searcher membaca `nonce = 0`; yang kedua **revert `InvalidNonce`**.
- Fee masuk 0,2 ETH per langkah. Searcher rasional melepas di langkah pertama yang untung setelah gas
  (0,005 ETH): wadah 10,0 ETH vs biaya 9,919 ETH → **bocor 80 bps** ke searcher.

## Pelajaran

1. **Efisiensi ditentukan kedalaman pool, bukan desain.** Buyback 10% pool membuang ~9% ke price impact di
   desain A; di desain B searcher menanggung impact yang sama lalu memasukkannya ke hitungan. Untuk riset
   tokenomics: bandingkan ukuran buyback mingguan dengan likuiditas pool token itu.
2. **Desain A memusatkan risiko di keeper** — kepercayaan, `minOut` off-chain ([EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)),
   penundaan, MEV. Satu `minOut = 0` = −29,5%.
3. **Desain B memindahkan risiko ke pasar** — tanpa keeper/oracle/swap; bocor ≈ gas + granularitas fee.
   Risikonya di **pengaturan `threshold`**: terlalu tinggi → rilis jarang dan tidak rata; terlalu rendah →
   gas memakan efisiensi. Itu sebabnya `setThreshold` dipegang governance (UNI: Timelock).
4. **Desain B membakar lebih banyak saat token murah.** Burn per rilis tetap, tapi harga GOV turun → biaya
   threshold turun → rilis lebih sering → lebih banyak GOV per ETH fee. Setara beli di harga pasar tanpa oracle.
5. **Desain B tidak pernah melakukan swap**, jadi kontraknya tidak bisa disandwich. Risiko MEV ditanggung
   searcher yang bersaing.

## Perbandingan dengan mesin nyata

| | CAKE (nyata) | Desain A | UNI Firepit (nyata) | Desain B |
| --- | --- | --- | --- | --- |
| Pemicu | bot EOA → Safe 3 tanda tangan → Timelock 7 hari | 1 keeper | siapa saja | siapa saja |
| Burn | kirim ke Safe, lalu `0xdead` mingguan | `burn()` langsung | `transferFrom` ke `0xdead` | `burnFrom` (kurangi supply) |
| `totalSupply` jujur | ❌ | ✅ | ❌ | ✅ |
| Threshold | — | — | 4.000 UNI | 90.000 GOV |
