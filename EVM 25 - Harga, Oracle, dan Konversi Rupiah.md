---
title: EVM 25 - Harga, Oracle, dan Konversi Rupiah
tags: [learning, evm, oracle, price, chainlink, amm]
updated: 2026-10-24
---

# Harga, Oracle, dan Konversi Rupiah

Bagian dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Catatan ini menjawab dua pertanyaan: harga di blockchain datanya dari mana?
Kalau wallet A punya 5 USDC dan mau dirupiahkan, harus pakai API pihak ketiga?

## Saldo bukan harga

`balanceOf(A)` di kontrak USDC hanya mengembalikan angka mentah, mis. `5000000`. USDC punya `decimals() = 6`,
jadi itu 5 USDC. Tidak ada harga, tidak ada mata uang fiat. Lihat [EVM 04 - ABI](EVM%2004%20-%20ABI.md) (selector `0x70a08231`).

## Dari mana "harga" di blockchain

Blockchain tidak punya harga bawaan. Ada dua sumber:

| Sumber | Cara kerja | Kelebihan | Kekurangan |
| --- | --- | --- | --- |
| **DEX / AMM pool** | Harga = rasio reserve pool, terbentuk dari trade on-chain ([EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md)) | Asli on-chain, tanpa pihak ketiga | Spot price bisa dimanipulasi dalam 1 blok (flash loan); pool tipis = harga jelek. Solusi: TWAP |
| **Oracle** (Chainlink, Pyth, dll.) | Node off-chain ambil harga dari exchange/data provider, agregasi, lalu tulis ke kontrak on-chain (Chainlink: `latestRoundData()` di aggregator) | Sulit dimanipulasi, sumber banyak | Ada trust ke jaringan oracle; update per heartbeat/deviation threshold, jadi bisa tertinggal |

Jadi oracle itu **jembatan data off-chain → on-chain**. Data mentahnya tetap dari exchange/API yang sama
yang bisa dipakai langsung di luar chain.

## Kenapa oracle ada

Smart contract tidak bisa melakukan HTTP call; ia hanya bisa membaca state chain. Kontrak yang butuh harga
(lending, liquidation, stablecoin, perp) harus membaca harga yang sudah ditulis ke chain → butuh oracle.
Bahaya kalau kontrak memakai spot price DEX: oracle manipulation ([EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md)).

## Merupiahkan 5 USDC

Tergantung **siapa yang butuh harganya**:

1. **Aplikasi off-chain (backend/frontend, tampilan saldo)** → pakai API biasa. Ini cara paling umum.
   - Kurs USD/IDR dari API forex, atau harga pasar USDT/IDR / USDC/IDR dari exchange (Indodax, Tokocrypto,
     Binance) atau aggregator (CoinGecko, CoinMarketCap).
   - Hitung: `5 × kurs`. USDC dianggap ≈ 1 USD; kalau mau presisi, kalikan juga harga USDC/USD.
   - Tidak perlu oracle: oracle datanya juga dari sumber seperti ini, dan lebih lambat update.
2. **Smart contract yang butuh nilai rupiah** → harus dari on-chain: feed oracle IDR (cek di
   data.chain.link apakah ada feed IDR untuk chain itu) atau pool DEX stablecoin rupiah vs USDC
   (likuiditasnya sering tipis → rawan manipulasi).
3. **Baca oracle dari backend** juga bisa (cukup `eth_call` lewat RPC, gratis), tapi tetap butuh RPC
   provider dan harganya hanya sesegar heartbeat feed.

Pilihan "IDR" yang dipakai juga beda arti: kurs forex resmi vs harga pasar kripto USDT/IDR bisa selisih
(premium/diskon lokal). Tentukan dulu angka mana yang dimaksud.

## Ringkas

- Saldo on-chain = angka token, bukan nilai.
- Harga on-chain datang dari DEX (terbentuk on-chain) atau oracle (dibawa dari off-chain).
- Untuk tampilan rupiah di aplikasi: **ya, pakai API pihak ketiga**. Oracle baru relevan kalau kontrak yang
  butuh harga.
