---
title: EVM 01.16 - Slashing
tags: [learning, evm, ethereum]
updated: 2026-10-10
---

# Slashing

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md)

Pertanyaan: slashing itu gimana cara kerjanya?

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
jaringan gagal mencapai finality selama lebih dari 4 epoch (*inactivity leak*). Bagian ini dari
pengetahuan umum.

## Kenapa perlu dihukum

Menandatangani itu gratis. Tanpa hukuman, validator bisa mendukung dua cabang rantai
sekaligus tanpa rugi apa pun, lalu tinggal ikut cabang yang menang (*nothing at stake*).
Slashing membuat tanda tangan ganda mahal. Akibatnya, dua blok final yang bertentangan cuma
mungkin kalau minimal 1/3 total stake rela dipotong.

## Alurnya

1. **Pelanggaran.** Validator menandatangani dua pesan bertentangan. Penyebab paling umum
   adalah kunci yang sama jalan di dua validator client (lihat [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md)).
2. **Ditemukan.** Node yang menjalankan *slasher* menyimpan pesan-pesan validator dan mencari
   pasangan yang bertentangan (pengetahuan umum).
3. **Dimasukkan ke blok.** Proposer berikutnya memasukkan bukti ke `attester_slashings` atau
   `proposer_slashings`. Protokol memeriksa kedua tanda tangan.
4. **Hukuman langsung.** Validator ditandai `slashed`, kena penalti awal, dan dijadwalkan keluar
   paksa. Proposer yang memasukkan bukti dapat reward pelapor.
5. **Masa tunggu 8.192 epoch (±36 hari).** Stake tidak bisa ditarik, dan validator tetap kena
   penalti seperti validator offline (pengetahuan umum).
6. **Penalti korelasi** di tengah masa tunggu. Makin banyak stake yang kena slashing dalam
   rentang ±36 hari yang sama, makin besar potongannya. Satu validator sendirian kena potongan
   hampir nol; kalau 1/3 jaringan kena bersamaan, seluruh stake hilang.
7. **Sisa saldo ditarik otomatis** ke alamat withdrawal setelah masa tunggu selesai.

Penalti korelasi dibuat begitu supaya kesalahan satu operator dihukum ringan, tapi serangan
terkoordinasi dihukum habis.

## Besarnya hukuman

Dari spesifikasi consensus (pengetahuan umum, **belum diukur**, karena saldo historis tidak
tersedia di node publik):

| Komponen | Sebelum Pectra (Mei 2025) | Sejak Pectra |
| --- | --- | --- |
| Penalti awal | 1/32 effective balance = **1 ETH** untuk 32 ETH | 1/4096 = **0,0078 ETH** |
| Reward pelapor (ke proposer) | 1/512 = 0,0625 ETH | 1/4096 = 0,0078 ETH |
| Penalti korelasi | effective balance × min(3 × stake yang kena slashing dalam ±36 hari ÷ total stake, 1) | prinsip sama |

## Kasus nyata: 17 validator, 21 Agustus 2026

| Waktu (UTC) | Kejadian |
| --- | --- |
| 02:39, slot 15.037.995–15.037.998 | 17 validator masing-masing memberi **dua** attestation untuk slot dan target epoch (469.937) yang sama, tapi memilih blok berbeda → double vote |
| 02:46, slot 15.038.030–15.038.047 | 17 blok berturut-turut masing-masing berisi tepat 1 `attester_slashing`, dimasukkan 17 proposer berbeda (slot 15.038.045 kosong, tidak ada blok) |
| 04:00, epoch 469.950 | validator 1.731.417 keluar paksa (`exit_epoch`) |
| 2026-09-26, epoch 478.130 | bisa ditarik: 469.938 + **8.192** epoch = 36,4 hari setelah dihukum. Sekarang saldonya 0 (sudah ditarik) |

Isi bukti di slot 15.038.030:

```text
attestation_1: slot 15.037.995, target epoch 469.937, blok 0xc2f7…  ← agregat 428 validator
attestation_2: slot 15.037.995, target epoch 469.937, blok 0x2a25…  ← cuma validator 1.731.417
irisan kedua daftar = [1.731.417]  → hanya dia yang dihukum
```

427 validator lain di attestation pertama cuma memilih sekali, jadi tidak dihukum.

Nomor ke-17 validator itu berdekatan (1.731.417–1.731.577), jadi kemungkinan besar milik
satu operator yang menjalankan kunci yang sama di dua tempat. Ini dugaan; pemilik dan penyebab
sebenarnya tidak dicek.

## Bukti (2026-10-10)

Data dari Beacon API publicnode.

| Cek | Hasil |
| --- | --- |
| Validator berstatus `active_slashed` / `exited_slashed` saat ini | 0 |
| Validator yang sudah keluar (`withdrawal_possible` + `withdrawal_done`) dengan `slashed` = true | **594** dari 1.527.047 (6 + 588) |
| Kelompok terbaru (withdrawable epoch 478.130–478.131) | 21 validator; 17 ditemukan di slot 15.038.030–15.038.047, 4 sisanya tidak dicari |
| Ke-17 `attester_slashings` | semuanya double vote: target epoch sama (469.937), slot attestation sama, `beacon_block_root` berbeda, satu korban per bukti |
| Saldo sebelum/sesudah dan endpoint `/eth/v1/beacon/rewards/blocks/15038030` | respons kosong: node publik tidak menyimpan state lama (bukan archive), jadi besar penalti tidak bisa diukur |

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.15 - Validator Client](EVM%2001.15%20-%20Validator%20Client.md)
