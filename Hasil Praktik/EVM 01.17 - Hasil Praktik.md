---
title: EVM 01.17 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.17 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.17 - Attestation](../EVM%2001%20-%20Detail/EVM%2001.17%20-%20Attestation.md). Dijalankan 2026-10-10 dengan data dari
Beacon API publicnode.

## Hasil

| Cek | Hasil |
| --- | --- |
| Komite epoch 481.227 (`/eth/v1/beacon/states/head/committees`) | **2.048 komite** = 32 slot × 64; ukuran 416–417; total **851.992** validator → setiap validator aktif muncul tepat sekali per epoch |
| Blok head slot 15.399.286: attestation gabungan pertama | untuk slot 15.399.285, mencakup **64 komite**, 26.632 bit, **26.570 suara (99,77%)**; head = `0xb9d6…` = blok slot 15.399.285; source epoch 481.226 → target 481.227; signature 96 byte |
| Attestation kedua di blok yang sama | 1 komite, cuma **2 suara**, head = `0x3407…` = blok slot 15.399.284 → dua validator ini belum melihat blok terbaru |
| Validator 2.092.386 | dijadwalkan di slot **15.399.266**, komite 0, posisi 290 |
| Suaranya di blok 15.399.267 (+1 slot) | bit ke-290 = **1** (ikut); head = `0x5d21…` = blok slot 15.399.266; target = `0xbb40…` = blok pertama epoch 481.227 (slot 15.399.264); source = epoch 481.226 |
| Attestation lain di blok 15.399.267 untuk slot yang sama | 16 komite memilih head `0x7a4e…` (blok slot 15.399.265) dan 2 komite memilih `0xbb40…` (slot 15.399.264): belum melihat blok slot 15.399.266. Validator 2.092.386 tidak ikut di gabungan ini |
| Finality saat dicek | epoch berjalan 481.227, justified 481.226, **finalized 481.225** → blok jadi final sekitar 2 epoch (±12,8 menit) di belakang |

## Yang tidak dicek

Bagian materi berikut berasal dari pengetahuan umum:

- Waktu memberi suara (sekitar 4 detik setelah slot dimulai, atau lebih cepat kalau blok sudah
  diterima).
- Peran *aggregator* di setiap komite.
- Langkah 5: aturan reward dan penalti attestation.
