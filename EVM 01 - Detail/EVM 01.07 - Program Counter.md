---
title: EVM 01.07 - Program Counter
tags: [learning, evm, pc]
updated: 2026-10-24
---

# Program Counter

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.06 - Storage](EVM%2001.06%20-%20Storage.md) · [EVM 01.08 - Gas](EVM%2001.08%20-%20Gas.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.07 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.07%20-%20Hasil%20Praktik.md).

Angka yang menunjuk **byte ke berapa di code yang sedang
dijalankan**, seperti jari yang menelusuri baris resep. Mulai dari 0. Setelah satu opcode
selesai, PC maju ke opcode berikutnya:

- Opcode biasa (1 byte) → PC + 1.
- `PUSH1`…`PUSH32` → PC + 1 + jumlah byte data. Byte data itu **bukan** instruksi dan dilewati.
- `JUMP` → PC diganti ke tujuan yang ada di stack. `JUMPI` → sama, tapi hanya kalau syaratnya
  bukan 0; kalau 0, PC + 1 seperti biasa. Inilah cara EVM membuat `if`, loop, dan memilih
  fungsi (dispatcher Solidity membandingkan selector lalu `JUMPI` ke kode fungsinya).
- Tujuan lompatan **wajib** byte `JUMPDEST` (`0x5b`) yang benar-benar opcode. Lompat ke tempat
  lain → `InvalidJump`, eksekusi gagal.

PC tidak bisa diubah langsung oleh kontrak (hanya lewat `JUMP`/`JUMPI`), dan tidak bertahan
setelah call selesai.

Contoh loop hitung mundur 3 → 0, code `0x60035b600190038060025700`:

```text
PC:    0  1    2     3  4    5     6    7     8  9    10     11
byte: 60 03   5b    60 01   90    03   80    60 02   57     00
      PUSH1 3 JUMPDEST PUSH1 1 SWAP1 SUB DUP1 PUSH1 2 JUMPI STOP
                 ▲                                     │
                 └──── kalau nilai ≠ 0, PC kembali ke 2 ┘
```

Jejak eksekusinya, seperti yang ditampilkan `cast run <tx> -t`. Stack ditulis **sebelum**
opcode dijalankan, elemen teratas di kanan:

| PC | Opcode | Stack | Catatan |
| ---: | --- | --- | --- |
| 0 | PUSH1 | `[]` | PC lompat ke 2 (lewati data `03`) |
| 2 | JUMPDEST | `[3]` | |
| 3 → 8 | PUSH1, SWAP1, SUB, DUP1, PUSH1 | … | 3 − 1 = 2 |
| 10 | JUMPI | `[2, 2, 2]` | tujuan 2, syarat 2 ≠ 0 → **PC = 2** |
| 2 | JUMPDEST | `[2]` | putaran ke-2 |
| 10 | JUMPI | `[1, 1, 2]` | syarat 1 ≠ 0 → PC = 2 |
| 2 | JUMPDEST | `[1]` | putaran ke-3 |
| 10 | JUMPI | `[0, 0, 2]` | syarat 0 → **tidak lompat**, PC = 11 |
| 11 | STOP | `[0]` | selesai, gas eksekusi 81 |

Lompatan ilegal, code `0x600456605b00` (`PUSH1 4 · JUMP · PUSH1 0x5b · STOP`): byte di PC 4
memang `0x5b`, tapi itu **data** milik `PUSH1` di PC 3, bukan opcode `JUMPDEST`. Hasilnya
`InvalidJump`, status transaksi 0, dan **seluruh gas limit habis**, karena kegagalan seperti ini
(beda dengan `REVERT`) memakan semua gas yang diberikan.

## Coba sendiri

1. Deploy code loop `0x60035b600190038060025700` sebagai kontrak di `anvil`, panggil, lalu
   lihat jejaknya dengan `cast run <tx> -t`. Cocokkan dengan tabel di atas.
2. Lakukan hal yang sama dengan code lompatan ilegal `0x600456605b00`, lalu cek status dan gas
   used di receipt.

```bash
cast run <tx> -t          # trace per opcode: PC, opcode, stack, sisa gas
cast receipt <tx>         # status + gasUsed
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.06 - Storage](EVM%2001.06%20-%20Storage.md) · [EVM 01.08 - Gas](EVM%2001.08%20-%20Gas.md) →
