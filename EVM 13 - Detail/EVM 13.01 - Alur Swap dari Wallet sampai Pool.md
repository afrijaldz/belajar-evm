---
title: EVM 13.01 - Alur Swap dari Wallet sampai Pool
tags: [learning, evm, defi, amm, uniswap, swap]
updated: 2026-10-24
---

# Alur Swap dari Wallet sampai Pool

Bagian dari [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) · terkait: [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) · [EVM 14 - Swap dari Kontrak dan Sandwich](../EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)

Hasil percobaan untuk materi ini ada di [EVM 13.01 - Hasil Praktik](../Hasil%20Praktik/EVM%2013.01%20-%20Hasil%20Praktik.md).

**Swap di DEX bukan "tukar dengan orang lain".** Tidak ada lawan transaksi dan tidak ada order
book. Yang ada: sebuah kontrak (pool/pair) yang memegang dua token. Kamu menyetor token A ke pool,
pool menghitung berapa token B yang boleh keluar supaya rumusnya tetap terjaga, lalu mengirim B ke
kamu. Semuanya terjadi di **satu transaksi** — kalau satu langkah gagal, semuanya revert.

Contoh di bawah memakai Uniswap v2 (kode paling sederhana); v3 dan PancakeSwap mengikuti pola
yang sama.

## Tiga pemain

| Pemain | Peran |
| --- | --- |
| **Wallet / frontend** | menghitung quote off-chain, memasang `amountOutMin` dan `deadline`, menandatangani transaksi |
| **Router** (`UniswapV2Router02`) | kontrak "pembantu": mencari alamat pair, menarik token kamu, memanggil pair, mengecek slippage |
| **Pair** (`UniswapV2Pair`) | kontrak yang benar-benar memegang reserve dan menegakkan `x·y=k` |

Router tidak memegang dana dan tidak wajib — kontrak bisa memanggil pair langsung. Router cuma
memudahkan dan menambah pengaman.

## Langkah demi langkah (swap 1.000 USDC → WETH)

### 0. Quote (off-chain, tidak ada transaksi)

Wallet membaca `getReserves()` atau memanggil `router.getAmountsOut(1000e6, [USDC, WETH])` lewat
`eth_call`. Hasilnya misal 0,3750 WETH. Dengan slippage 0,5%:
`amountOutMin = 0,3750 × 0,995 = 0,3731 WETH`.

### 1. `approve` (transaksi pertama, sekali saja)

Token ERC-20 tidak bisa "didorong" ke kontrak sambil memanggil fungsinya. Jadi kamu memberi izin:
`USDC.approve(router, 1000e6)`. Ini yang muncul sebagai "Approve USDC" di wallet. Risikonya ada di
[EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) (unlimited approve, permit phishing). Swap dari ETH native
tidak butuh langkah ini — ETH dikirim sebagai `msg.value`, router membungkusnya jadi WETH.

### 2. Panggil router (transaksi kedua)

```solidity
router.swapExactTokensForTokens(
    1000e6,             // amountIn
    0.3731e18,          // amountOutMin
    [USDC, WETH],       // path
    msg.sender,         // to
    block.timestamp + 20 minutes  // deadline (diisi wallet)
);
```

Di dalam router:

1. `require(deadline >= block.timestamp)` → kalau lewat: `EXPIRED`.
2. Hitung `amounts = getAmountsOut(amountIn, path)` — **dari reserve saat transaksi dieksekusi**,
   bukan saat quote.
3. `require(amounts[last] >= amountOutMin)` → kalau kurang: `INSUFFICIENT_OUTPUT_AMOUNT`. Ini
   pengaman slippage/sandwich.
4. `transferFrom(kamu → pair, 1000 USDC)` — memakai allowance dari langkah 1. Token masuk
   **langsung ke pair**, tidak mampir di router.
5. Panggil `pair.swap(amount0Out, amount1Out, to, "")`.

### 3. Di dalam pair: kirim dulu, cek belakangan

`UniswapV2Pair.swap` bekerja "optimistic":

1. Kirim token keluar dulu (`0,375 WETH → to`).
2. Kalau `data` tidak kosong, panggil `uniswapV2Call` di penerima — ini **flash swap** (pinjam dulu,
   bayar di callback yang sama).
3. Baca `balanceOf(this)` untuk kedua token. Token yang masuk dihitung dari selisih saldo vs reserve
   lama: `amountIn = balance − (reserve − amountOut)`. Pair tidak peduli siapa yang mengirim atau
   kapan, yang penting saldonya sudah ada sebelum cek ini.
4. **Cek invariant dengan fee 0,3%:**

```
(balance0 × 1000 − amount0In × 3) × (balance1 × 1000 − amount1In × 3) ≥ reserve0 × reserve1 × 1000²
```

   Kalau gagal → `UniswapV2: K`, seluruh transaksi revert (termasuk transfer keluar di langkah 1).
5. Simpan reserve baru (`_update`), emit event `Swap`.

Karena fee 0,3% dipotong dari sisi input sebelum cek, `k` setelah swap sedikit lebih besar — itu
pendapatan LP (lihat [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) bagian "Fee tertinggal di pool").

### 4. Selesai

WETH sudah di wallet. Di explorer, satu transaksi swap biasanya berisi log: `Transfer` USDC (kamu →
pair), `Transfer` WETH (pair → kamu), `Sync` (reserve baru), `Swap`.

## Kenapa harga bisa "jelek"

Harga tidak ditentukan siapa pun — murni dari rumus. Makin besar `amountIn` dibanding reserve, makin
jauh titiknya bergeser di kurva `x·y=k` → **price impact**. Di [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md): swap kecil
kena sekitar sebesar fee, swap seperempat reserve kena sekitar 20%.

Selisih antara quote (langkah 0) dan eksekusi (langkah 2) adalah **slippage**: transaksi lain —
termasuk penyerang sandwich — bisa masuk duluan dan menggeser reserve. `amountOutMin` membatasi
seberapa jauh itu boleh terjadi ([EVM 14 - Swap dari Kontrak dan Sandwich](../EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)).

## Multi-hop

Kalau tidak ada pair langsung, `path = [TOKEN, WETH, USDC]`. Router menghitung tiap hop, mengirim
input ke pair pertama, dan pair pertama mengirim outputnya **langsung ke pair kedua** (`to` = pair
berikutnya). Hanya pair terakhir yang mengirim ke kamu. Tiap hop kena fee dan price impact sendiri.

## Variasi yang sering ditemui

| Jenis | Bedanya dengan v2 |
| --- | --- |
| **Uniswap v3 / PancakeSwap v3** | likuiditas per rentang harga (tick); pool memanggil `uniswapV3SwapCallback` dan **kamu/router membayar di callback** — bukan transfer duluan. Lihat [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](../EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md) |
| **Uniswap v4** | semua pool di satu kontrak `PoolManager`; saldo dicatat sebagai utang (flash accounting) dan diselesaikan di akhir |
| **Aggregator** (1inch, 0x, Paraswap) | mencari rute termurah lintas banyak DEX, bisa memecah order ke beberapa pool; ujungnya tetap memanggil pool-pool di atas |
| **Intent / RFQ** (UniswapX, CoW Swap) | kamu menandatangani pesanan off-chain; *solver*/market maker yang mengeksekusi dan mengisi dari mana saja. Di sini memang ada lawan transaksi |
| **Curve / StableSwap** | rumus berbeda (lebih datar di sekitar 1:1) untuk pasangan stablecoin, price impact jauh lebih kecil |

## Coba sendiri: `mini-amm`

Cara paling cepat memahami alur di atas: tulis Uniswap v2 versi kecil dari nol — `Token`,
`MiniPair`, `MiniFactory`, `MiniRouter` — dan jalankan lokal tanpa RPC/fork.

1. Buat pool test, misalnya 1.000 WETH + 2.000.000 USDC (2.000 USDC/WETH).
2. Swap 1.000 USDC lewat router. Bandingkan output dengan quote, hitung harga efektif dan harga spot
   baru, dan pastikan router tidak memegang token setelah swap.
3. Swap langsung ke pair tanpa router. Minta output 1 wei lebih dari yang boleh → harus revert `K`.
4. Transfer token ke pair di transaksi terpisah, lalu lihat bahwa orang lain bisa mengambil
   output-nya. Ini alasan transfer dan `swap` harus dalam satu transaksi.
5. Jual 1 / 10 / 100 / 500 WETH dan catat price impact tiap ukuran.
6. Bolak-balik swap beberapa kali, lalu bandingkan kerugian trader dengan tambahan yang ditarik LP.
7. Multi-hop tiga token, flash swap (bayar balik dengan fee vs tanpa fee), dan sandwich dengan dan
   tanpa slippage.
8. Deploy ke `anvil` dan jalankan swap manual dengan `cast`. `cast run <tx>` menampilkan urutan
   `transferFrom` → `pair.swap` → `transfer` keluar → `balanceOf` → `Sync` → `Swap`, sama dengan
   langkah 2–3 di atas.

```bash
forge test -vv                  # semua test + log
forge test --mt test_01 -vvvv   # trace lengkap satu test swap
cast run <tx>                   # urutan panggilan satu swap di anvil
```

Latihan lanjutan:

1. Ubah fee jadi 0,25% (gaya PancakeSwap).
2. Tambahkan swap exact output.
3. Apa yang terjadi kalau pair mempercayai angka dari caller, bukan saldonya sendiri?
4. Hapus `lock` (reentrancy guard) dan cari cara menyalahgunakannya.
5. Hitung titik break-even sandwich.

## Ringkasan satu kalimat

Swap = kirim token A ke kontrak pool → pool hitung output dari reserve dengan rumus `x·y=k` dikurangi
fee → pool kirim token B → pool cek rumus masih terpenuhi; router membungkus itu dengan `approve`,
`deadline`, dan `amountOutMin`.

---

Bagian dari [EVM 13 - AMM x·y=k](../EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) · terkait: [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) · [EVM 14 - Swap dari Kontrak dan Sandwich](../EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md)
