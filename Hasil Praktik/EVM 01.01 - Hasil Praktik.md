---
title: EVM 01.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.01 - Mesin EVM](../EVM%2001%20-%20Detail/EVM%2001.01%20-%20Mesin%20EVM.md). Dijalankan 2026-10-03 di `anvil` lokal.

## Program 2 + 3 → storage slot 0

Program `0x600260030160005500` di-deploy sebagai kontrak lalu dipanggil.

- `cast run <tx> -vvvvv` → `storage changes: @ 0: 0 → 5`.
- Gas eksekusi `[22112]`.
- Total `gasUsed` 43.112 = 21.000 (biaya dasar transaksi) + 22.112.

Kolom stack di tabel materi diturunkan dari definisi opcode, dan total gasnya cocok dengan hasil
di atas. Trace per langkah bisa dilihat dengan `cast run <tx> -t`.
