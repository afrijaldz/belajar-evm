---
title: EVM 16 - Buyback-and-Burn Sendiri
tags: [learning, evm, defi, tokenomics, buyback, burn]
updated: 2026-10-24
---

# Buyback-and-Burn Sendiri

Sesi 16 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu menulis dua desain buyback-and-burn sendiri dan
mengujinya di fork Ethereum dengan token `GOV` buatan sendiri + pool Uniswap v2 asli. Desainnya meniru dua
mesin yang sudah dibongkar: pipeline CAKE ([EVM 04 - ABI](EVM%2004%20-%20ABI.md), [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)) dan Firepit UNI
([EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)). Lanjutan dari [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md).

Hasil percobaan untuk materi ini ada di [EVM 16 - Hasil Praktik](Hasil%20Praktik/EVM%2016%20-%20Hasil%20Praktik.md).

## Setup

- `GovToken`: `burn()`/`burnFrom()` **mengurangi `totalSupply`** (gaya A, [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md)) —
  supply selalu jujur, tidak perlu rumus supply efektif.
- Pool Uniswap v2 GOV/ETH sebagai pasar.
- Fee protokol terkumpul dalam WETH. Tugas mesin buyback: mengubah fee itu menjadi GOV yang terbakar.

## Dua desain

| | A — `BuybackBurner` (gaya CAKE) | B — `FeeAuction` (gaya UNI Firepit) |
| --- | --- | --- |
| Siapa beli | keeper terpercaya | siapa saja (searcher) |
| Cara | swap WETH → GOV di AMM, `burn` hasilnya | bayar `threshold` GOV (dibakar), ambil seluruh isi wadah |
| Butuh AMM/oracle di kontrak | ya | tidak |
| Pengaman | `minGovOut` dari off-chain + `deadline` | `nonce` (hanya satu pemanggil menang) |

### Desain A di bawah sandwich

Desain A melakukan swap, jadi transaksinya bisa disandwich ([EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)).
Penyerang front-run sebelum transaksi keeper:

- Dengan `minGovOut = 0`, swap tetap jalan di harga yang sudah dipompa. GOV yang terbakar turun jauh dan
  penyerang mengambil selisihnya sebagai profit.
- Dengan `minGovOut` dari quote off-chain (mis. −0,5%), swap **revert** `INSUFFICIENT_OUTPUT_AMOUNT`. Tidak ada
  yang terbakar di harga buruk.

### Desain B: persaingan dan kebocoran

- Beberapa searcher bisa membaca `nonce` yang sama. Hanya yang pertama masuk blok yang menang; yang lain
  **revert `InvalidNonce`**.
- Fee masuk ke wadah sedikit demi sedikit. Searcher rasional melepas wadah di langkah pertama yang untung
  setelah gas. Selisih antara isi wadah dan harga `threshold` itulah yang **bocor** ke searcher.

## Pelajaran

1. **Efisiensi ditentukan kedalaman pool, bukan desain.** Buyback yang besar dibanding pool membuang banyak
   nilai ke price impact — buyback senilai 10% isi pool kehilangan sekitar 9%. Di desain A protokol yang
   menanggung impact itu; di desain B searcher menanggung impact yang sama lalu memasukkannya ke hitungan.
   Untuk riset tokenomics: bandingkan ukuran buyback mingguan dengan likuiditas pool token itu.
2. **Desain A memusatkan risiko di keeper** — kepercayaan, `minOut` off-chain ([EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)),
   penundaan, MEV. Satu `minOut = 0` bisa memotong burn hampir sepertiganya.
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

## Coba sendiri

1. Tulis `GovToken` dengan `burn()`/`burnFrom()` yang mengurangi `totalSupply`.
2. Di fork Ethereum, buat pool Uniswap v2 **1.000.000 GOV + 100 ETH** (1 GOV = 0,0001 ETH) dan siapkan fee
   protokol **10 WETH**.
3. Jalankan desain A dan desain B (`threshold` 90.000 GOV). Bandingkan GOV terbakar dan GOV per ETH fee.
4. Sandwich desain A: front-run 20 ETH sebelum transaksi keeper, sekali dengan `minGovOut = 0` dan sekali
   dengan quote off-chain −0,5%.
5. Desain B: biarkan dua searcher membaca `nonce = 0`. Lalu masukkan fee 0,2 ETH per langkah dan cari
   langkah pertama yang untung setelah gas (anggap gas 0,005 ETH).

```bash
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```
