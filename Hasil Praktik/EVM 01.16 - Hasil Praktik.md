---
title: EVM 01.16 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.16 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.16 - Slashing](../EVM%2001%20-%20Detail/EVM%2001.16%20-%20Slashing.md). Dijalankan 2026-10-10 dengan data dari
Beacon API publicnode.

## Hasil

| Cek | Hasil |
| --- | --- |
| Validator berstatus `active_slashed` / `exited_slashed` saat ini | 0 |
| Validator yang sudah keluar (`withdrawal_possible` + `withdrawal_done`) dengan `slashed` = true | **594** dari 1.527.047 (6 + 588) |
| Kelompok terbaru (withdrawable epoch 478.130–478.131) | 21 validator; 17 ditemukan di slot 15.038.030–15.038.047, 4 sisanya tidak dicari |
| Ke-17 `attester_slashings` | semuanya double vote: target epoch sama (469.937), slot attestation sama, `beacon_block_root` berbeda, satu korban per bukti |
| Saldo sebelum/sesudah dan endpoint `/eth/v1/beacon/rewards/blocks/15038030` | respons kosong: node publik tidak menyimpan state lama (bukan archive), jadi besar penalti tidak bisa diukur |

## Kasus nyata: 17 validator, 21 Agustus 2026

| Waktu (UTC) | Kejadian |
| --- | --- |
| 02:39, slot 15.037.995–15.037.998 | 17 validator masing-masing memberi **dua** attestation untuk slot dan target epoch (469.937) yang sama, tapi memilih blok berbeda → double vote |
| 02:46, slot 15.038.030–15.038.047 | 17 blok berturut-turut masing-masing berisi tepat 1 `attester_slashing`, dimasukkan 17 proposer berbeda (slot 15.038.045 kosong, tidak ada blok) |
| 04:00, epoch 469.950 | validator 1.731.417 keluar paksa (`exit_epoch`) |
| 2026-09-26, epoch 478.130 | bisa ditarik: 469.938 + **8.192** epoch = 36,4 hari setelah dihukum. Saat dicek saldonya 0 (sudah ditarik) |

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

## Yang tidak dicek

Bagian materi berikut berasal dari pengetahuan umum:

- Penalti offline dan *inactivity leak*.
- Langkah 2 (cara kerja *slasher*) dan langkah 5 (penalti selama masa tunggu).
- Tabel "Besarnya hukuman" (dari spesifikasi consensus). **Belum diukur**, karena saldo
  historis tidak tersedia di node publik.
