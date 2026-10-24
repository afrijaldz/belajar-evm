---
title: EVM 20 - Gas Optimization
tags: [learning, evm, solidity, gas, foundry]
updated: 2026-10-24
---

# Gas Optimization

Sesi 20 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Target: `Vault.sol` dari [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) — kontrak dengan
jaring pengaman terlengkap (17 test, 3 invariant, cabang 100% dari [EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md)). Satu
perubahan per langkah; setiap langkah wajib lolos semua test.

Hasil percobaan untuk materi ini ada di [EVM 20 - Hasil Praktik](Hasil%20Praktik/EVM%2020%20-%20Hasil%20Praktik.md).

```bash
python3 bench.py                                              # gas transaksi nyata, skenario tetap
forge snapshot --no-match-contract VaultInvariantTest --diff  # dibanding snapshot tersimpan
forge test --gas-report                                       # per fungsi (dalam test: cold/warm campur)
```

`bench.py` di sini adalah script kecil buatan sendiri: deploy vault ke `anvil`, jalankan skenario tetap
(deposit pertama, deposit lagi, withdraw), lalu catat gas tiap transaksi nyata. `.gas-snapshot` adalah
baseline dari `forge snapshot`.

## Metode

1. Baseline dulu: `forge snapshot` + gas report + `bench.py`.
2. Satu perubahan → semua test + invariant → `bench.py`.
3. Angka yang dipercaya = **transaksi nyata**, bukan gas di dalam test (pelajaran [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md),
   [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md)).

Langkah yang dicoba berurutan: optimizer on → `abi.encodeCall` → `unchecked` pada pengurangan yang aman →
menghapus `totalDeposits` → `via_ir`. Ukur deploy dan setiap jenis transaksi (deposit pertama vs berikutnya,
withdraw) di tiap langkah.

## Pelajaran

1. **Optimizer = kemenangan gratis terbesar untuk deploy** (bisa hampir setengahnya). Foundry mematikannya
   secara bawaan. `optimizer_runs` tinggi → runtime lebih murah, deploy lebih mahal; rendah → sebaliknya.
2. **Runtime didominasi storage dan panggilan eksternal.** Compiler cuma bisa memotong beberapa persen.
3. **Tips populer bisa bernilai nol.** `abi.encodeWithSignature("…")` → `abi.encodeCall`: selisih **0 gas**,
   karena optimizer sudah menghitung keccak string literal saat kompilasi. Tetap pakai `encodeCall` karena
   dicek tipenya oleh compiler.
4. **`unchecked` hanya dengan bukti.** Hematnya kecil (ratusan gas per pengurangan). Setiap blok `unchecked`
   diberi komentar *kenapa* aman dan invariant mana yang menjaminnya.
5. **Pengungkit terbesar adalah desain: jumlah slot storage.** Satu slot baru = 20.000 gas
   ([EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)), jadi menghapus variabel seperti `totalDeposits` memotong deposit pertama
   sekitar sepertiganya. Tapi `totalDeposits` memungkinkan siapa pun mengecek solvency on-chain dan dipakai
   invariant. Untuk vault yang memegang dana orang lain, auditabilitas biasanya sepadan dengan harganya.
   Alternatifnya pembukuan lewat event (murah, [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md)) tapi hanya bisa dicek
   off-chain.
6. **`via_ir`**: deploy jauh lebih murah lagi, runtime cuma hemat beberapa ratus gas, kompilasi lebih lambat.

## Yang sudah otomatis optimal dari sesi sebelumnya

- Custom error, bukan `require` string ([EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md)).
- `immutable` untuk alamat yang tidak berubah ([EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md)).
- Mapping bersarang, bukan array yang di-loop.
- Event untuk riwayat, storage hanya untuk keadaan sekarang.

## Coba sendiri

1. Buat baseline: `forge snapshot`, `forge test --gas-report`, dan benchmark transaksi nyata di `anvil`.
2. Nyalakan optimizer, jalankan semua test, ukur lagi.
3. Ganti `abi.encodeWithSignature` dengan `abi.encodeCall`; bandingkan gas.
4. Bungkus pengurangan yang terbukti aman dengan `unchecked` (beri komentar alasannya).
5. Coba hapus `totalDeposits` dan coba `via_ir`, ukur, lalu putuskan apakah layak dipakai.
6. Akhiri dengan `forge snapshot --diff` untuk melihat perubahan per test.
