---
title: EVM 01.17 - Attestation
tags: [learning, evm, ethereum]
updated: 2026-10-10
---

# Attestation

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) · [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md) →

Pertanyaan: attestation itu gimana cara kerjanya?

**Attestation adalah suara yang ditandatangani validator.** Isinya kira-kira: "menurut saya,
ujung rantai sekarang adalah blok X, dan saya mendukung checkpoint Y menuju Z." Setiap
validator aktif wajib memberi **satu** attestation per epoch, di slot yang sudah dijadwalkan.

Suara ini dipakai untuk dua hal:

1. **Memilih ujung rantai (fork choice).** Kalau ada dua cabang, node mengikuti cabang dengan
   dukungan suara terbanyak.
2. **Finality.** Kalau minimal 2/3 total stake mendukung checkpoint yang sama, checkpoint itu
   jadi *justified*. Dua checkpoint berturut-turut justified → yang lebih awal jadi
   *finalized* dan tidak bisa dibatalkan tanpa 1/3 stake kena slashing (detailnya di
   [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md)).

## Isi satu attestation

| Field | Arti | Dipakai untuk |
| --- | --- | --- |
| `slot` | slot jadwal validator ini | |
| `beacon_block_root` | blok yang menurut validator adalah ujung rantai sekarang (*head vote*) | fork choice |
| `source` | checkpoint terakhir yang sudah justified | finality |
| `target` | checkpoint epoch ini = blok pertama epoch ini | finality |
| `signature` | tanda tangan BLS 96 byte | |

`source` → `target` adalah "link" antara dua checkpoint. Aturan slashing di
[EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) (double vote, surround vote) berlaku untuk pasangan ini.

## Komite: siapa memberi suara kapan

Semua validator aktif diacak lalu dibagi rata ke 32 slot × 64 komite setiap epoch. Jadi
setiap slot, sekitar 1/32 validator memberi suara.

```text
Epoch (32 slot, 6,4 menit)
├── slot 0  → 64 komite × ±416 validator → ±26.600 suara
├── slot 1  → 64 komite × ±416 validator
│   ...
└── slot 31 → 64 komite × ±416 validator
            = setiap validator aktif tepat sekali per epoch
```

Validator memberi suara sekitar 4 detik setelah slot dimulai, atau lebih cepat kalau blok slot
itu sudah diterima (pengetahuan umum). Validator yang belum melihat blok terbaru akan memilih
blok terakhir yang ia lihat. Ini tidak dihukum, cuma reward head-nya hilang.

## Agregasi: puluhan ribu suara jadi satu

Tanda tangan BLS bisa **digabung**. Ribuan tanda tangan untuk isi yang sama digabung jadi satu
tanda tangan 96 byte, ditambah `aggregation_bits` (satu bit per anggota komite: 1 = ikut
memberi suara). Beberapa validator per komite ditunjuk sebagai *aggregator* untuk
menggabungkan (pengetahuan umum).

Sejak Pectra (Mei 2025), satu attestation gabungan bisa mencakup **semua 64 komite** sekaligus
(`committee_bits`). Hasilnya, satu blok cukup berisi beberapa attestation untuk menampung
puluhan ribu suara.

## Alurnya untuk satu validator

1. Validator client bertanya ke beacon node: "saya memberi suara di slot mana?" (lihat
   [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md)).
2. Saat slotnya tiba: tentukan ujung rantai, isi `source` dan `target`, tanda tangani.
3. Suara disebar ke jaringan dan digabung aggregator.
4. Proposer slot berikutnya memasukkan attestation gabungan ke bloknya.
5. Reward diberikan kalau `source`, `target`, dan `head` benar dan suara masuk cepat. Kalau
   salah atau terlambat, kena penalti kecil (pengetahuan umum).

## Bukti (2026-10-10)

Data dari Beacon API publicnode.

| Cek | Hasil |
| --- | --- |
| Komite epoch 481.227 (`/eth/v1/beacon/states/head/committees`) | **2.048 komite** = 32 slot × 64; ukuran 416–417; total **851.992** validator → setiap validator aktif muncul tepat sekali per epoch |
| Blok head slot 15.399.286: attestation gabungan pertama | untuk slot 15.399.285, mencakup **64 komite**, 26.632 bit, **26.570 suara (99,77%)**; head = `0xb9d6…` = blok slot 15.399.285; source epoch 481.226 → target 481.227; signature 96 byte |
| Attestation kedua di blok yang sama | 1 komite, cuma **2 suara**, head = `0x3407…` = blok slot 15.399.284 → dua validator ini belum melihat blok terbaru |
| Validator 2.092.386 | dijadwalkan di slot **15.399.266**, komite 0, posisi 290 |
| Suaranya di blok 15.399.267 (+1 slot) | bit ke-290 = **1** (ikut); head = `0x5d21…` = blok slot 15.399.266; target = `0xbb40…` = blok pertama epoch 481.227 (slot 15.399.264); source = epoch 481.226 |
| Attestation lain di blok 15.399.267 untuk slot yang sama | 16 komite memilih head `0x7a4e…` (blok slot 15.399.265) dan 2 komite memilih `0xbb40…` (slot 15.399.264): belum melihat blok slot 15.399.266. Validator 2.092.386 tidak ikut di gabungan ini |
| Finality saat dicek | epoch berjalan 481.227, justified 481.226, **finalized 481.225** → blok jadi final sekitar 2 epoch (±12,8 menit) di belakang |

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) · [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md) →
