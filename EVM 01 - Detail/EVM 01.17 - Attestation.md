---
title: EVM 01.17 - Attestation
tags: [learning, evm, ethereum]
updated: 2026-10-24
---

# Attestation

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) · [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.17 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.17%20-%20Hasil%20Praktik.md).

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

Semua validator aktif diacak lalu dibagi rata ke 32 slot × 64 komite = **2.048 komite** setiap
epoch. Jadi setiap slot, sekitar 1/32 validator memberi suara. Ukuran satu komite = jumlah
validator aktif ÷ 2.048; dengan sekitar 850 ribu validator aktif, satu komite berisi ±416
validator.

```text
Epoch (32 slot, 6,4 menit)
├── slot 0  → 64 komite × ±416 validator → ±26.600 suara
├── slot 1  → 64 komite × ±416 validator
│   ...
└── slot 31 → 64 komite × ±416 validator
            = setiap validator aktif tepat sekali per epoch
```

Validator memberi suara sekitar 4 detik setelah slot dimulai, atau lebih cepat kalau blok slot
itu sudah diterima. Validator yang belum melihat blok terbaru akan memilih
blok terakhir yang ia lihat. Ini tidak dihukum, cuma reward head-nya hilang.

## Agregasi: puluhan ribu suara jadi satu

Tanda tangan BLS bisa **digabung**. Ribuan tanda tangan untuk isi yang sama digabung jadi satu
tanda tangan 96 byte, ditambah `aggregation_bits` (satu bit per anggota komite: 1 = ikut
memberi suara). Beberapa validator per komite ditunjuk sebagai *aggregator* untuk
menggabungkan.

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
   salah atau terlambat, kena penalti kecil.

## Yang terlihat di blok

- **Suara slot N masuk di blok slot N+1.** Proposer berikutnya yang memasukkannya.
- **Satu attestation gabungan bisa berisi puluhan ribu suara** dari 64 komite, dengan tingkat
  partisipasi biasanya di atas 99%.
- **Tidak semua suara memilih head yang sama.** Di blok yang sama sering ada attestation kecil
  yang memilih blok lebih lama. Validator di dalamnya belum melihat blok terbaru saat memberi
  suara.
- **`target` selalu blok pertama epoch berjalan, `source` checkpoint epoch sebelumnya.**
- **Finality tertinggal sekitar 2 epoch.** Saat epoch N berjalan, checkpoint N-1 biasanya
  justified dan N-2 finalized, jadi blok jadi final sekitar ±12,8 menit di belakang (lihat
  [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md)).

## Coba sendiri

1. Ambil daftar komite satu epoch, lalu hitung jumlah komite dan total validator.
2. Buka blok terbaru dan lihat attestation di dalamnya: `aggregation_bits`, `committee_bits`,
   `beacon_block_root`, `source`, `target`.
3. Cari jadwal satu validator (slot, komite, posisi), lalu cek bit posisinya di attestation
   pada blok slot berikutnya.
4. Lihat `finality_checkpoints` untuk membandingkan epoch berjalan dengan justified dan
   finalized.

```bash
BEACON=https://ethereum-beacon-api.publicnode.com
curl -s "$BEACON/eth/v1/beacon/states/head/committees?epoch=<epoch>" | jq '.data | length'
curl -s $BEACON/eth/v2/beacon/blocks/head | jq '.data.message.body.attestations[0].data'
curl -s $BEACON/eth/v1/beacon/states/head/finality_checkpoints
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) · [EVM 01.18 - Finality](EVM%2001.18%20-%20Finality.md) →
