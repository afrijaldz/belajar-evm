---
title: EVM 01.16 - Slashing
tags: [learning, evm, ethereum]
updated: 2026-10-24
---

# Slashing

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md) · [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.16 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.16%20-%20Hasil%20Praktik.md).

**Slashing adalah hukuman otomatis dari protokol untuk validator yang terbukti menandatangani
dua pesan yang bertentangan.** Buktinya adalah dua pesan itu sendiri, lengkap dengan tanda
tangan si validator. Siapa pun yang menemukan bukti itu bisa memasukkannya ke blok. Protokol
memeriksa tanda tangannya, lalu memotong stake dan mengeluarkan validator itu secara paksa.

Tidak ada yang memutuskan secara manual. Kalau kedua tanda tangan sah dan isinya bertentangan,
hukuman pasti jalan.

## Tiga pelanggaran yang dihukum

| Pelanggaran | Artinya | Bukti di beacon block |
| --- | --- | --- |
| **Double proposal** | membuat dua blok berbeda untuk slot yang sama | `proposer_slashings`: dua header blok yang ditandatangani |
| **Double vote** | memberi dua attestation berbeda untuk target epoch yang sama | `attester_slashings`: dua attestation |
| **Surround vote** | attestation yang "mengurung" attestation lain (source lebih awal **dan** target lebih akhir) | `attester_slashings`: dua attestation |

**Offline bukan slashing.** Validator yang mati atau telat cuma kehilangan reward dan kena
penalti kecil, kira-kira sebesar reward yang hilang. Penalti offline baru membesar kalau
jaringan gagal mencapai finality selama lebih dari 4 epoch (*inactivity leak*).

## Kenapa perlu dihukum

Menandatangani itu gratis. Tanpa hukuman, validator bisa mendukung dua cabang rantai
sekaligus tanpa rugi apa pun, lalu tinggal ikut cabang yang menang (*nothing at stake*).
Slashing membuat tanda tangan ganda mahal. Akibatnya, dua blok final yang bertentangan cuma
mungkin kalau minimal 1/3 total stake rela dipotong.

## Alurnya

1. **Pelanggaran.** Validator menandatangani dua pesan bertentangan. Penyebab paling umum
   adalah kunci yang sama jalan di dua validator client (lihat [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md)).
2. **Ditemukan.** Node yang menjalankan *slasher* menyimpan pesan-pesan validator dan mencari
   pasangan yang bertentangan.
3. **Dimasukkan ke blok.** Proposer berikutnya memasukkan bukti ke `attester_slashings` atau
   `proposer_slashings`. Protokol memeriksa kedua tanda tangan.
4. **Hukuman langsung.** Validator ditandai `slashed`, kena penalti awal, dan dijadwalkan keluar
   paksa. Proposer yang memasukkan bukti dapat reward pelapor.
5. **Masa tunggu 8.192 epoch (±36 hari).** Stake tidak bisa ditarik, dan validator tetap kena
   penalti seperti validator offline.
6. **Penalti korelasi** di tengah masa tunggu. Makin banyak stake yang kena slashing dalam
   rentang ±36 hari yang sama, makin besar potongannya. Satu validator sendirian kena potongan
   hampir nol; kalau 1/3 jaringan kena bersamaan, seluruh stake hilang.
7. **Sisa saldo ditarik otomatis** ke alamat withdrawal setelah masa tunggu selesai.

Penalti korelasi dibuat begitu supaya kesalahan satu operator dihukum ringan, tapi serangan
terkoordinasi dihukum habis.

## Besarnya hukuman

Dari spesifikasi consensus:

| Komponen | Sebelum Pectra (Mei 2025) | Sejak Pectra |
| --- | --- | --- |
| Penalti awal | 1/32 effective balance = **1 ETH** untuk 32 ETH | 1/4096 = **0,0078 ETH** |
| Reward pelapor (ke proposer) | 1/512 = 0,0625 ETH | 1/4096 = 0,0078 ETH |
| Penalti korelasi | effective balance × min(3 × stake yang kena slashing dalam ±36 hari ÷ total stake, 1) | prinsip sama |

## Seperti apa bukti double vote

Satu `attester_slashing` berisi dua attestation. Keduanya punya slot dan target epoch yang sama,
tapi memilih blok berbeda. Attestation pertama biasanya agregat dari ratusan validator;
attestation kedua bisa cuma dari satu validator.

```text
attestation_1: slot S, target epoch E, blok A  ← agregat ratusan validator
attestation_2: slot S, target epoch E, blok B  ← cuma validator V
irisan kedua daftar = [V]  → hanya V yang dihukum
```

Yang dihukum hanya validator yang ada di **irisan** kedua daftar. Validator lain di attestation
pertama cuma memilih sekali, jadi tidak dihukum.

Kalau banyak validator dengan nomor berdekatan kena slashing pada waktu yang sama, kemungkinan
besar mereka milik satu operator yang menjalankan kunci yang sama di dua tempat. Proposer yang
berbeda-beda memasukkan bukti-bukti itu ke blok yang berurutan. Sekitar 8.192 epoch setelah
dihukum, saldo yang tersisa bisa ditarik.

Kasus nyata dengan 17 validator dibahas di [EVM 01.16 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.16%20-%20Hasil%20Praktik.md).

## Coba sendiri

1. Cari validator yang sedang dalam status `active_slashed` atau `exited_slashed`.
2. Cari validator yang sudah keluar dengan `slashed` = true.
3. Buka blok yang berisi `attester_slashings`, lalu bandingkan dua attestation di dalamnya.

```bash
BEACON=https://ethereum-beacon-api.publicnode.com
curl -s "$BEACON/eth/v1/beacon/states/head/validators?status=active_slashed,exited_slashed"
curl -s "$BEACON/eth/v1/beacon/states/head/validators?status=withdrawal_possible,withdrawal_done" \
  | jq '[.data[] | select(.validator.slashed)] | length'
curl -s $BEACON/eth/v2/beacon/blocks/<slot> | jq '.data.message.body.attester_slashings'
```

**Jebakan:** node publik biasanya bukan node arsip, jadi saldo validator sebelum dan sesudah
dihukum (dan endpoint `/eth/v1/beacon/rewards/blocks/<slot>` untuk slot lama) tidak tersedia.
Besar penalti dari kasus lama tidak bisa diukur lewat node seperti itu.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md) · [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md) →
