---
title: EVM 14 - Swap dari Kontrak dan Sandwich
tags: [learning, evm, defi, uniswap, mev, slippage]
updated: 2026-10-24
---

# Swap dari Kontrak dan Sandwich

Sesi 14 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar melakukan swap dari kontrak lewat router
Uniswap v2, dua pengaman yang disediakan router, cara kerja serangan sandwich, dan kenapa
`amountOutMin` harus dihitung off-chain. Lanjutan dari [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md).

Hasil percobaan untuk materi ini ada di [EVM 14 - Hasil Praktik](Hasil%20Praktik/EVM%2014%20-%20Hasil%20Praktik.md).

## Dua pengaman di router

| Parameter | Revert kalau | Melindungi dari |
| --- | --- | --- |
| `deadline` | `block.timestamp > deadline` → `UniswapV2Router: EXPIRED` | transaksi yang ditahan di mempool / oleh validator sampai harga berubah |
| `amountOutMin` | hasil < min → `UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT` | harga bergerak sebelum transaksi dieksekusi, termasuk sandwich |

## Sandwich

Urutan dalam satu blok: **penyerang beli → korban beli → penyerang jual**. Pembelian penyerang
menaikkan harga, korban membeli di harga lebih buruk dan mendorong harga lebih jauh, penyerang
menjual di harga tertinggi itu.

### Tanpa proteksi (`amountOutMin = 0`)

Kerugian korban tidak terbatas: makin besar modal front-run penyerang, makin banyak yang diambil.
Di pool yang cukup dangkal dibanding ukuran swap korban, korban bisa kehilangan puluhan persen
dari hasil swap-nya.

### Slippage dihitung off-chain (sebelum kirim)

Wallet menghitung `amountOutMin = quote × (1 − slippage)` sebelum mengirim transaksi, misalnya
quote × 0,995 untuk slippage 0,5%.

- Front-run kecil: korban tetap lolos, tapi kehilangan sampai hampir batas slippage.
- Front-run yang mendorong hasil di bawah `amountOutMin`: **transaksi korban revert**.
- **Slippage tolerance = batas maksimal yang rela diserahkan ke penyerang.** Penyerang yang optimal
  mengambil hampir sampai batas itu.
- Di dunia nyata penyerang mengirim front-run + back-run sebagai **bundle atomik**: kalau korban
  revert, bundle tidak dieksekusi — penyerang tidak rugi. Korban cuma kehilangan gas.

### "Proteksi" palsu: quote di dalam transaksi

Kontrak yang menghitung `amountOutMin` dari `getAmountsOut` **di dalam** transaksi tidak terlindungi
sama sekali. Quote itu diambil **setelah** front-run menggeser harga, jadi hasilnya sama dengan
`amountOutMin = 0`. `amountOutMin` harus dihitung off-chain sebelum mengirim, seperti wallet.

## Pelajaran

1. Jangan pernah `amountOutMin = 0` di mainnet (aman hanya di fork).
2. Slippage sekecil yang masih wajar untuk ukuran order dan likuiditas pool.
3. Quote on-chain dari pool yang sama bukan pengaman.
4. Deadline pendek; jangan `type(uint256).max`.
5. Order besar di pool dangkal: kirim lewat mempool privat (mis. Flashbots Protect) atau pecah.

## Jebakan Foundry

`vm.prank(attacker)` + `call{value: x}` **tidak memotong ETH dari `attacker`** — `msg.sender`
berganti, tapi saldo attacker tidak turun. Kalau profit penyerang dihitung dari `attacker.balance`,
hasilnya ikut menghitung modal front-run. Hitung profit = ETH hasil jual − ETH front-run, dari
return value swap.

## Coba sendiri

1. Tulis kontrak `SwapHelper` yang melakukan swap lewat router Uniswap v2, dengan `deadline` dan
   `amountOutMin` sebagai parameter.
2. Test bahwa deadline yang sudah lewat dan `amountOutMin` di atas quote sama-sama revert.
3. Simulasikan sandwich di fork Ethereum (pool WETH/USDC): korban swap 50 ETH → USDC, penyerang
   front-run dengan berbagai ukuran. Ulangi tanpa proteksi, dengan slippage 0,5% yang dihitung
   sebelum transaksi, dan dengan quote on-chain.

```bash
forge test --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
```
