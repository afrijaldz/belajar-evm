---
title: EVM 19 - Testing Lanjutan
tags: [learning, evm, foundry, testing, fuzz, invariant, coverage]
updated: 2026-10-24
---

# Testing Lanjutan

Sesi 19 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md) — sesi pertama Fase 4. Dasar fuzz/invariant/fork sudah dipakai di
[EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md), [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md), [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md). Di sesi ini
kamu belajar coverage, differential testing, ghost variable, shrinking, dan fork test yang rapi. Lanjutan dari
[EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md).

Hasil percobaan untuk materi ini ada di [EVM 19 - Hasil Praktik](Hasil%20Praktik/EVM%2019%20-%20Hasil%20Praktik.md).

## 1. Coverage: baris 100% bisa menipu

```bash
forge coverage --no-match-contract VaultInvariantTest --report summary
forge coverage --no-match-contract VaultInvariantTest --report lcov    # detail per cabang (BRDA)
```

**Coverage baris** hanya bertanya "apakah baris ini pernah jalan?". **Coverage cabang** bertanya "apakah setiap
`if` pernah diuji ke dua arah?". Kontrak bisa 100% baris tapi hanya 40% cabang: setiap baris pernah jalan,
tapi banyak `if … revert` hanya pernah diuji satu arah. Lihat laporan `lcov` untuk tahu cabang mana yang
belum disentuh.

Coverage cabang juga bisa menemukan **pengecekan mati**: `if` yang tidak pernah bisa tercapai karena kode
sebelumnya sudah revert duluan. Contoh: cek `token.code.length == 0` yang letaknya **setelah** panggilan
`balanceOf(token)`. Pada alamat tanpa kode, `balanceOf` sudah revert saat decode, jadi cek itu tidak pernah
jalan. Perbaikannya: pindahkan cek ke awal fungsi.

**Jebakan:** `vm.expectRevert()` **tanpa argumen** lolos oleh revert apa pun. Ia tidak membedakan revert yang
dirancang dari revert kebetulan, jadi test bisa lolos karena alasan yang salah. Selalu sebutkan error yang
diharapkan, mis. `vm.expectRevert(abi.encodeWithSelector(TokenCallFailed.selector, notAToken))`.

Untuk menutup cabang yang belum diuji, tulis test untuk kasus pinggir: deposit 0, withdraw 0, ETH ke kontrak
yang menolak, token yang revert, token yang return `false` (pakai mock seperti `FalseReturnToken`).

## 2. Differential testing

Differential test membandingkan implementasimu dengan **implementasi referensi** untuk input acak. Ia tidak
butuh jawaban benar per kasus — cukup referensinya. Untuk rumus AMM, referensinya bisa router Uniswap v2 asli
di fork (`getAmountOut` adalah fungsi `pure`).

| Versi | Rumus |
| --- | --- |
| `getAmountOut` (sama dengan router) | `in×997×R_out / (R_in×1000 + in×997)` |
| `getAmountOutEarlyFee` | `f = in×997/1000; f×R_out / (R_in + f)` |

Di atas kertas dua rumus ini identik, tapi versi kedua **membulatkan dua kali**: pembagian `/1000` dibulatkan
dulu, lalu hasilnya dipakai lagi. Untuk input wajar (mis. 1 ETH ke pool biasa) hasilnya sama persis, jadi test
manual lolos. Untuk input kecil atau reserve yang timpang, selisihnya bisa sangat besar — input 1 wei bahkan
bisa menghasilkan 0. Fuzz menemukan kasus itu; test manual tidak.

**Test manual menguji kasus yang terpikir; fuzz menguji yang tidak terpikir.**

## 3. Ghost variable dan shrinking

**Ghost variable** adalah catatan yang dibuat handler invariant sendiri, di luar kontrak. Contoh: handler
mencatat token yang benar-benar masuk/keluar vault (`ghostIn`, `ghostOut`, diukur dari selisih saldo).
Invariant-nya: `totalDeposits == ghostIn − ghostOut` — lebih tajam dari sekadar "vault solvent".

Untuk membuktikan invariant ini punya gigi, buat mutation: mis. `withdraw` lupa mengurangi `totalDeposits`
saat saldo ditarik penuh. Invariant harus gagal dengan pesan seperti `books != real flows`.

**Shrinking:** saat invariant gagal, Foundry memangkas urutan panggilan acak yang panjang menjadi urutan
terpendek yang masih gagal (mis. cukup `deposit`, lalu `withdraw` seluruh saldo). Urutan pendek ini langsung
menunjuk bug. Urutan yang gagal disimpan di `cache/invariant/failures/` dan diulang duluan di run berikutnya.

`afterInvariant()` dipanggil setelah tiap run — tempat mencetak statistik handler (jumlah deposit/withdraw).

## 4. Fork test yang rapi

```toml
# foundry.toml
[rpc_endpoints]
mainnet = "https://eth.drpc.org"
```

```solidity
function setUp() public {
    vm.createSelectFork("mainnet", 26_046_500); // blok di-pin: hasil sama di setiap run
}
```

Data fork di-cache per blok di `~/.foundry/cache/rpc/mainnet/<blok>/`. Blok yang di-pin = test deterministik
+ cepat: run pertama mengambil data dari RPC, run berikutnya membaca cache dan jauh lebih cepat. Tanpa pin,
setiap run memakai blok terbaru dan hasilnya bisa berubah.

## Ringkasan teknik

| Teknik | Menjawab | Kapan dipakai |
| --- | --- | --- |
| Unit test | kasus yang terpikir | selalu |
| Fuzz | input acak untuk satu fungsi | fungsi matematis, batas nilai |
| Invariant + handler | urutan panggilan acak, multi-aktor | kontrak yang punya state |
| Ghost variable | "apakah pembukuan = aliran nyata?" | vault, lending, AMM |
| Differential | "apakah sama dengan referensi?" | port rumus, optimasi gas |
| Coverage cabang | "cabang mana belum pernah diuji?" | sebelum audit |
| Mutation | "apakah test bisa gagal?" | membuktikan test punya gigi |

## Coba sendiri

1. Jalankan `forge coverage` pada `Vault.sol` dari [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md). Bandingkan persen baris dan
   cabang, lalu cari cabang yang belum diuji di laporan `lcov`.
2. Cari test yang memakai `vm.expectRevert()` kosong dan perketat ke error spesifik. Cek apakah test itu masih
   lolos.
3. Tulis `getAmountOut` dan `getAmountOutEarlyFee`, lalu fuzz keduanya terhadap router Uniswap v2 di fork.
4. Tambahkan `ghostIn`/`ghostOut` ke handler invariant vault, lalu buat mutation di `withdraw` dan lihat hasil
   shrinking.
5. Pin blok fork di `setUp`, lalu bandingkan waktu run pertama dan kedua.
