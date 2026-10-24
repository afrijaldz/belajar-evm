---
title: EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi
tags: [learning, evm, defi, uniswap, clmm]
updated: 2026-10-24
---

# Uniswap v3: Likuiditas Terkonsentrasi

Sesi 15 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar cara membaca pool Uniswap v3 (harga
sebagai akar, tick, likuiditas aktif), kenapa v3 jauh lebih dalam dari v2 dengan TVL yang sama, dan
untung-rugi posisi likuiditas sempit vs lebar. Lanjutan dari [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md).

Hasil percobaan untuk materi ini ada di [EVM 15 - Hasil Praktik](Hasil%20Praktik/EVM%2015%20-%20Hasil%20Praktik.md).

## Membaca pool

Setiap pool v3 punya pasangan token, fee tier (mis. 500 = 0,05%), dan `tickSpacing` (10 untuk fee
0,05%). State utamanya:

| Field | Arti |
| --- | --- |
| `slot0.sqrtPriceX96` | `√harga × 2⁹⁶` (fixed-point Q64.96) |
| `slot0.tick` | `floor(log₁.₀₀₀₁(harga))` — bisa dihitung ulang dari `sqrtPriceX96` |
| `liquidity` | `L` yang aktif di rentang tick sekarang saja |

```
harga mentah = (sqrtPriceX96 / 2⁹⁶)²             → token1 per token0, dalam satuan terkecil
USDC/ETH     = 1 / (harga mentah × 10^(6 − 18))   → untuk pool token0 USDC, token1 WETH
```

Satu tick = perubahan harga 0,01%. Rentang aktif adalah satu `tickSpacing` di sekitar tick
sekarang — untuk `tickSpacing` 10, lebarnya sekitar 0,1%. Seluruh `L` aktif bekerja di rentang itu.
Harga disimpan sebagai **akar** karena di dalam satu tick rumus swap linear terhadap `√P`.

## v3 vs v2: kedalaman

Di v2, likuiditas tersebar merata dari harga 0 sampai tak hingga. Di v3, LP memilih rentang, dan
sebagian besar likuiditas menumpuk dekat harga sekarang. Akibatnya pool v3 dengan TVL beberapa kali
lebih besar bisa memberi price impact **belasan kali lebih kecil** untuk swap besar — selisihnya
adalah efek konsentrasi.

Swap besar di v3 melewati banyak tick. Quote untuk swap yang sangat besar bisa gagal dalam satu
`eth_call` karena terlalu banyak tick yang harus dilewati.

## Posisi likuiditas

`NonfungiblePositionManager.mint` membuat posisi — tiap posisi = NFT ERC-721. Ujung rentang harus
kelipatan `tickSpacing`.

Dengan modal yang sama, makin sempit rentangnya, makin besar `L` yang didapat. Posisi ±0,5% bisa
punya `L` per dolar ratusan kali posisi penuh; posisi penuh (0–∞) setara dengan v2.

### Fee: dibagi sesuai porsi `L`

Selama harga di dalam rentang, fee dibagi sesuai porsi `L` yang aktif. Rasio fee antar posisi ≈
rasio `L`-nya, jadi posisi sempit mendapat fee jauh lebih besar.

### Harga keluar rentang

Kalau harga turun melewati batas bawah, posisi sempit **sudah menjual semua token quote (mis. USDC)
untuk membeli token base (mis. ETH) selama harga turun** — isinya 100% ETH. Posisi lebar dan penuh
masih berisi campuran kedua token.

Di luar rentang, posisi **tidak dapat fee sama sekali** sampai harga kembali. Untuk dapat fee lagi
harus rebalance (tarik, swap, buat baru) — ada biaya. Konsentrasi = fee lebih besar **dan** risiko
keluar rentang lebih besar.

## Jebakan

- **Estimasi gas bisa kurang untuk panggilan bersarang (aturan 63/64, EIP-150).** Swap lewat router
  bisa revert `OutOfGas` walau limit dari estimasi. Wallet menambah margin di atas estimasi; di
  script pakai `--gas-limit` eksplisit.
- **`cast send` tanpa `--json` tidak selalu gagal saat transaksi revert.** Selalu cek `status`
  receipt.
- `cast keccak` itu perintah lokal — menolak `--rpc-url`.
- State fork **menumpuk antar percobaan**: setiap run script yang gagal di tengah tetap meninggalkan
  swap yang sudah terjadi, dan harga ikut bergeser. Restart `anvil` atau pakai `evm_snapshot` /
  `evm_revert`.

## Coba sendiri

Jalankan `anvil --fork-url https://eth.drpc.org` di port 8545. Pool contoh: WETH/USDC 0,05%
`0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640`.

1. Baca `slot0`, `liquidity`, `token0`, `fee`, `tickSpacing`. Hitung harga USDC/ETH dari
   `sqrtPriceX96`, dan hitung ulang tick-nya.
2. Bandingkan price impact swap 1, 100, dan 1.000 ETH di v3 (`QuoterV2.quoteExactInputSingle`) dan
   v2 (`getAmountsOut`) di blok yang sama.
3. Buat tiga posisi dengan modal sama: sempit ±0,5%, lebar ±50%, dan penuh. Bandingkan `L`-nya.
4. Swap bolak-balik beberapa kali dengan harga tetap di dalam rentang sempit, lalu bandingkan fee
   tiap posisi (`collect` statis).
5. Jual WETH dalam jumlah besar sampai harga keluar rentang sempit, lalu bandingkan isi tiap posisi
   (`decreaseLiquidity` statis).

```bash
cast call <pool> 'slot0()(uint160,int24,uint16,uint16,uint16,uint8,bool)'
cast call <pool> 'liquidity()(uint128)'
cast call <pool> 'tickSpacing()(int24)' ; cast call <pool> 'fee()(uint24)'
cast send <router> ... --gas-limit <n> --json      # cek field status di receipt
```
