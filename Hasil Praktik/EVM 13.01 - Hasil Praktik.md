---
title: EVM 13.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 13.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 13.01 - Alur Swap dari Wallet sampai Pool](../EVM%2013%20-%20Detail/EVM%2013.01%20-%20Alur%20Swap%20dari%20Wallet%20sampai%20Pool.md). Project dibuat
2026-10-18, jalan lokal tanpa RPC/fork.

Project: `~/Documents/riset/evm/mini-amm` — Uniswap v2 versi kecil yang ditulis dari nol: `Token`,
`MiniPair`, `MiniFactory`, `MiniRouter`.

```bash
cd ~/Documents/riset/evm/mini-amm
forge test -vv                  # 13 test + log
forge test --mt test_01 -vvvv   # trace lengkap satu swap
```

## Hasil test

Pool test: 1.000 WETH + 2.000.000 USDC (2.000 USDC/WETH).

| Test | Hasil |
| --- | --- |
| `test_01` swap 1.000 USDC | dapat 0,4982 WETH = quote; harga efektif 2.007,02; spot jadi 2.001,997; router tidak memegang token |
| `test_05` / `test_06` | swap langsung ke pair tanpa router berhasil; minta 1 wei lebih → `MiniPair: K` |
| `test_07` | transfer ke pair di transaksi terpisah → orang lain yang mengambil output-nya |
| `test_08` jual 1 / 10 / 100 / 500 WETH | price impact 0,39% / 1,28% / 9,33% / 33,46% |
| `test_09` 5× bolak-balik 10.000 USDC | trader rugi 294,55 USDC = persis tambahan yang ditarik LP |
| `test_10` USDC → WETH → PEPE | output pair 1 langsung masuk pair 2 |
| `test_11` / `test_12` flash swap 100 WETH | bayar balik 100,3009 lolos; bayar 100 → `K` |
| `test_13` sandwich front-run 200.000 USDC | korban 4,10 bukan 4,96 WETH, penyerang +637,58 USDC; dengan slippage 0,5% korban revert |

## Swap manual di anvil

Swap manual di anvil (`script/Deploy.s.sol` + `cast`) juga sudah dicoba: `cast run <tx>` menampilkan
urutan `transferFrom` → `pair.swap` → `transfer` keluar → `balanceOf` → `Sync` → `Swap`, sama
dengan langkah 2–3 di materi. Perintahnya dan 5 latihan (fee Pancake, exact output, mempercayai
caller, hapus `lock`, sandwich break-even) ada di `README.md` project.

## Dokumentasi project

Dokumentasi lengkap project (Inggris) ada di `~/Documents/riset/evm/mini-amm/docs/` — 11 halaman:
arsitektur, lifecycle swap dengan referensi baris, matematika (`getAmountOut`, cek `K`, price impact,
fee flash swap), liquidity, multi-hop + flash swap, security, referensi kontrak, test, playground
anvil, perbedaan dengan Uniswap v2, latihan. Mulai dari `docs/README.md`.
