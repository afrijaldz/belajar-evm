---
title: EVM 13 - AMM x·y=k
tags: [learning, evm, defi, amm, uniswap, pancakeswap]
updated: 2026-10-24
---

# AMM x·y=k

Sesi 13 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md) — sesi pertama Fase 3. Di sesi ini kamu belajar cara kerja AMM
`x·y=k` ala Uniswap v2 dan PancakeSwap v2: harga dari reserve, rumus swap, price impact, cara LP
dibayar, dan cara fee protokol diambil. Lanjutan dari [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md).

Hasil percobaan untuk materi ini ada di [EVM 13 - Hasil Praktik](Hasil%20Praktik/EVM%2013%20-%20Hasil%20Praktik.md).

## Harga = rasio reserve

Sebuah pair memegang dua token. `getReserves()` mengembalikan jumlah keduanya, dan **harga spot =
reserve token B / reserve token A**. Contoh pair WETH/USDC: kalau reserve-nya 10.000.000 USDC dan
4.000 WETH, harga spot = 10.000.000 / 4.000 = **$2.500 / ETH**.

`token0` / `token1` diurutkan berdasarkan alamat token, bukan berdasarkan nama. Di pair WETH/USDC,
`token0` adalah USDC.

Tidak ada order book. Harga sepenuhnya ditentukan dua angka `getReserves()`.

## Rumus swap

Fee 0,3% ditahan, lalu `x·y` dijaga:

```
amountOut = (Δx × 997 × y) / (x × 1000 + Δx × 997)
```

Router (`getAmountsOut`) memakai rumus yang sama, jadi hasil rumus sendiri, quote router, dan swap
nyata cocok sampai satuan terkecil — selama state pool sama.

### Penjelasan detail

1. [EVM 13.01 - Alur Swap dari Wallet sampai Pool](EVM%2013%20-%20Detail/EVM%2013.01%20-%20Alur%20Swap%20dari%20Wallet%20sampai%20Pool.md) — alur lengkap satu swap: quote off-chain, `approve`, router (`deadline`, `amountOutMin`, `transferFrom` ke pair), `pair.swap` kirim-dulu-cek-belakangan, cek invariant + fee, multi-hop, variasi v3/v4/aggregator/intent/Curve

## Price impact

Makin besar input dibanding reserve, makin jauh titiknya bergeser di kurva, dan makin buruk harga
efektifnya. Kenaikannya tidak linear:

- Swap kecil (di bawah 0,1% reserve): price impact ≈ fee 0,3%.
- Swap ~2,5% reserve: sekitar −2,8%.
- Swap ~25% reserve: sekitar **−20%**.

Itu sebabnya likuiditas pool perlu dibandingkan dengan ukuran posisi (dan dengan market cap token):
pool dangkal menghukum posisi berukuran sedang.

## Fee tertinggal di pool → k naik

`k` naik sedikit setelah tiap swap, makin besar untuk swap yang makin besar. Fee tidak dibagikan per
swap; ia **menambah reserve**, dan token LP mewakili pool yang makin besar. Itu cara LP dibayar.

## Fee protokol: dicetak sebagai LP ke `feeTo`

Uniswap v2 fee switch = 1/6 dari 0,3% (≈17%). Tidak ada transfer per swap: pair menghitung
pertumbuhan `√k` sejak `kLast`, lalu **mencetak LP** untuk `feeTo` saat ada `mint`/`burn` likuiditas:

```
liquidity = totalSupply × (√k − √kLast) / (5 × √k + √kLast)      // UniswapV2Pair._mintFee
```

Penerima fee protokol menyimpan fee sebagai **token LP**. Untuk mengambil token dasarnya, penerima
memicu `mint` (yang menjalankan `_mintFee`) lalu `burn` (menukar LP jadi token dasar). Di Uniswap v2,
`feeTo` adalah TokenJar — itu sebabnya transaksi bot di [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)
berisi log `Mint` lalu `Burn` di puluhan pair.

## PancakeSwap v2 (BSC)

PancakeSwap v2 memakai rumus yang sama, tapi fee **0,25%**:

```
amountOut = (Δx × 9975 × y) / (x × 10000 + Δx × 9975)
```

Cara membuktikan fee sebuah fork Uniswap: hitung output dengan beberapa rumus fee, lalu lihat mana
yang persis sama dengan `getAmountsOut` router-nya.

Harga dalam USD bisa diturunkan dari reserve: harga token dalam BNB × harga BNB. Pancake v2 juga
punya `factory.feeTo()`, yang menerima fee protokol Pancake v2.

## Coba sendiri

Fork Ethereum dengan `anvil --fork-url https://eth.drpc.org` di port 8545. Untuk PancakeSwap, baca
langsung ke RPC BSC.

1. Baca `getReserves()` dan `token0()` pair WETH/USDC `0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc`,
   lalu hitung harga spot.
2. Hitung output swap 1, 10, 100, dan 1.000 ETH dengan rumus sendiri, bandingkan dengan
   `getAmountsOut`, lalu lakukan swap nyata. Mulai tiap swap dari state yang sama
   (`evm_snapshot` / `evm_revert`).
3. Bandingkan `k` sebelum dan sesudah tiap swap.
4. Baca `factory.feeTo()` dan `kLast()`, lalu tambah likuiditas dan lihat event `Mint` LP ke `feeTo`.
5. Di pair CAKE/WBNB `0x0eD7e52944161450477ee417DE9Cd3a859b14fD0`, bandingkan rumus 9975/10000 dan
   997/1000 dengan router Pancake.

```bash
cast call <pair> 'getReserves()(uint112,uint112,uint32)'
cast call <pair> 'token0()(address)' ; cast call <pair> 'kLast()(uint256)'
cast call <factory> 'feeTo()(address)' ; cast call <factory> 'getPair(address,address)(address)' <a> <b>
cast call <router> 'getAmountsOut(uint256,address[])(uint256[])' <amount> "[<in>,<out>]"
cast send <router> 'swapExactETHForTokens(uint256,address[],address,uint256)' 0 "[<weth>,<token>]" <to> <deadline> --value <wei>
```

`amountOutMin = 0` di perintah terakhir hanya aman di fork. Di mainnet, itu mengundang sandwich
([EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)).
