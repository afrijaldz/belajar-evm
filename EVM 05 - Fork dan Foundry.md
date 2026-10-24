---
title: EVM 05 - Fork dan Foundry
tags: [learning, evm, bsc, foundry, fork]
updated: 2026-10-24
---

# Fork dan Foundry EVM

Sesi 5 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), lanjutan dari [EVM 04 - ABI](EVM%2004%20-%20ABI.md). Di sesi ini kamu belajar
menjalankan salinan lokal chain asli (fork), lalu memakainya untuk tiga hal: menyamar sebagai
akun lain, menulis storage langsung, dan menjalankan test Foundry di atas state nyata.

Hasil percobaan untuk materi ini ada di [EVM 05 - Hasil Praktik](Hasil%20Praktik/EVM%2005%20-%20Hasil%20Praktik.md).

## Apa itu fork

```bash
anvil --fork-url <rpc-arsip> --fork-block-number <blok>
```

- Chain lokal yang **meminjam state chain asli** di blok tertentu. Chain ID tetap sama dengan
  chain aslinya (BSC = 56).
- State diambil **lazy**: hanya slot yang disentuh yang diminta ke RPC, lalu di-cache.
- Semua perubahan hanya lokal. Akun `anvil` otomatis dapat 10.000 koin native palsu (BNB di fork
  BSC).
- **Pin nomor blok** supaya hasil bisa diulang persis. Biasanya dipilih satu blok sebelum
  transaksi yang ingin kamu ulang.

## Syarat utama: archive node

Fork di blok lama butuh RPC **archive**. Full node hanya menyimpan state ~128 blok terakhir:
di BSC (blok 0,45 detik) itu **kurang dari 1 menit**, di Ethereum (12 detik) ~25 menit.

RPC gratis sering tidak cocok untuk fork. Masalah yang umum:

- bukan full archive (`state at block … is pruned`, `missing trie node`);
- hanya mengizinkan metode EVM inti, sehingga metode yang dipakai `anvil` ditolak;
- tidak mendukung query historis seperti `eth_getBalance` di blok lama;
- archive hanya untuk pengguna dengan token pribadi;
- rate limit (HTTP 429). Untuk RPC seperti ini pasang `--compute-units-per-second` rendah.

Gejala RPC tidak cocok: transaksi **pending selamanya** (`txpool_status` → `pending: 0x1`) walau
`cast call` masih jalan. Ini bisa terjadi kalau `anvil` gagal mengambil state akun yang dibutuhkan
untuk memproses blok (misalnya akun validator) — semua transaksi macet, termasuk transfer polos.
Cek dengan transfer polos dulu sebelum menyalahkan kontraknya.

## Cara 1: menyamar (impersonate)

Di fork, siapa pun bisa jadi siapa pun. Misalnya untuk memanggil fungsi `onlyOwner` atas nama
owner kontrak:

```bash
cast rpc anvil_impersonateAccount <owner>
cast rpc anvil_setBalance <owner> 0x8AC7230489E80000    # 10 BNB untuk gas
cast send <kontrak> "burnCake(bool)" false --from <owner> --unlocked
```

Tanpa menyamar: revert `Ownable: caller is not the owner` — datanya diawali `0x08c379a0`,
selector `Error(string)`. Pesan revert juga di-encode ABI (lihat [EVM 04 - ABI](EVM%2004%20-%20ABI.md)).

Fork yang di-pin satu blok sebelum transaksi asli, lalu menjalankan panggilan yang sama, akan
menghasilkan perpindahan token yang sama persis dengan transaksi asli. Ini cara memastikan kamu
benar-benar memahami sebuah mekanisme.

**Jebakan:** fungsi yang menyentuh ribuan slot (misalnya loop update semua pool) bisa **membuat
fork macet** di RPC publik, karena setiap slot yang belum di-cache harus diambil satu per satu dari
RPC. Kalau ada versi fungsi yang lebih ringan, pakai itu.

## Cara 2: menulis storage

Untuk memberi saldo token ke akun mana pun, tulis langsung slot mapping `balances`. Caranya
mencari nomor slot mapping: hitung `keccak256(alamat, i)` untuk holder yang saldonya diketahui
dengan `i` = 0, 1, 2, …, lalu cocokkan dengan `balanceOf`. Index yang cocok = slot mapping.

```bash
cast rpc anvil_setStorageAt <token> $(cast index address <saya> <slot>) $(cast to-uint256 1000000000000000000000)
```

**Menulis storage melewati semua logika dan bisa merusak invariant.** Saldo bertambah, transfer
jalan, tapi `totalSupply` tidak berubah — jumlah semua saldo kini lebih besar dari `totalSupply`.
Cheatcode `deal()` Foundry melakukan hal yang sama.

`evm_snapshot` / `evm_revert` mengembalikan state seketika — pakai untuk mengulang skenario.

## Cara 3: test Foundry di atas fork

```bash
forge test --match-contract <NamaTest> \
  --fork-url <rpc-arsip> --fork-block-number <blok> --compute-units-per-second 20 -vv
```

- `vm.prank(<alamat>)` = impersonate untuk satu panggilan.
- `vm.expectRevert("<pesan>")` = memastikan panggilan revert dengan pesan tertentu, misalnya
  untuk membuktikan hanya owner yang boleh memanggil.
- Test seperti ini bisa membuktikan klaim tentang protokol nyata (misalnya "hanya Timelock yang
  bisa burn") dengan state asli, dan bisa diulang kapan saja karena bloknya di-pin.

## Jebakan: sampling berkala

Saldo kontrak bisa naik-turun dalam siklus (misalnya menumpuk sepanjang minggu lalu dikosongkan
seminggu sekali). Kalau saldo diambil secara berkala dengan interval lebih panjang dari siklusnya,
titik sampel bisa selalu jatuh di "lembah" atau "puncak" siklus. **Sampling berkala bisa tertipu
siklus yang lebih pendek dari intervalnya.** Fork di beberapa blok berbeda membantu melihat
siklus ini.

## Jebakan lain

- **Checksum alamat (EIP-55).** Solidity menolak alamat dengan huruf besar-kecil salah. Verifikasi
  dengan `cast to-check-sum-address`. Hati-hati regex 40 hex juga menangkap potongan hash transaksi.
- **`emit` tidak boleh di fungsi `view`** — log adalah perubahan state.
- `cast send` menunggu konfirmasi dengan timeout. Untuk fork lambat: `--async`, lalu poll
  `cast receipt`.
- `pkill -f <pola>` bisa membunuh shell sendiri; pakai trik `[b]urn_retry.sh` atau `pkill -x anvil`.

## Coba sendiri

1. Cari RPC archive untuk chain yang ingin di-fork, lalu jalankan `anvil --fork-url` dengan
   `--fork-block-number`.
2. Kirim transfer polos untuk memastikan fork tidak macet.
3. Pilih kontrak dengan fungsi `onlyOwner`. Panggil tanpa menyamar (harus revert), lalu dengan
   `anvil_impersonateAccount`.
4. Cari slot mapping `balances` sebuah token dengan `cast index`, tulis saldo dengan
   `anvil_setStorageAt`, lalu bandingkan jumlah saldo dengan `totalSupply`.
5. Tulis test Foundry dengan `vm.prank` dan `vm.expectRevert`, jalankan dengan `--fork-url`.
