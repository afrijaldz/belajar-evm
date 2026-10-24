---
title: EVM 01.15 - Validator Client
tags: [learning, evm, ethereum]
updated: 2026-10-24
---

# Validator Client

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md) · [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.15 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.15%20-%20Hasil%20Praktik.md).

**Validator client adalah program terpisah.** Biasanya ia jalan di komputer yang sama dengan
consensus client (beacon node) dan terhubung ke beacon node lewat Beacon API. Ia **tidak**
terhubung ke execution client.

Tugasnya kecil tapi paling sensitif: memegang **kunci tanda tangan validator**, menanyakan
jadwal tugas ke beacon node, lalu menandatangani blok dan suara (attestation) saat jadwalnya
tiba. Semua pekerjaan berat (menjalankan EVM, menyimpan state, ikut jaringan P2P) dikerjakan
dua client lainnya.

## Posisinya di satu mesin staker

```text
┌──────────────────────────── satu komputer staker rumahan ─────────────────────────────┐
│                                                                                       │
│  Execution client  ◄── Engine API ──►  Consensus client  ◄── Beacon API ──  Validator │
│  (geth, reth, ...)                     (beacon node)                        client    │
│  EVM, state, mempool                   PoS, fork choice                     kunci BLS │
│        │                                     │                              slashing  │
│        ▼                                     ▼                              protection│
│   P2P execution                         P2P consensus                     (tanpa P2P) │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

Satu validator client bisa memegang banyak kunci validator sekaligus (satu kunci = satu
validator 32 ETH).

## Yang dikerjakan validator client

1. Setiap epoch (32 slot = 6,4 menit) bertanya ke beacon node: "validator saya dapat tugas
   apa?" lewat `/eth/v1/validator/duties/proposer/{epoch}` dan endpoint attester.
2. Setiap epoch, setiap validator wajib memberi satu **attestation** (suara untuk blok yang
   dianggap benar, lihat [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)).
3. Kalau dapat giliran membuat blok: minta beacon node menyiapkan blok (beacon node meminta
   isinya ke execution client, lihat [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md)),
   tanda tangani, lalu kirim kembali ke beacon node untuk disebar.
4. Menyimpan **slashing protection database**: catatan semua yang sudah ditandatangani, supaya
   tidak pernah menandatangani dua hal yang bertentangan.

Jadwal pembuat blok dari endpoint di langkah 1 berisi 32 entri per epoch, satu pembuat blok
per slot. Slot pertama epoch N adalah N × 32. Beacon block di slot itu nanti punya
`proposer_index` yang sama dengan jadwalnya.

## Kenapa dipisah dari beacon node

- **Kunci terisolasi.** Program yang memegang kunci dibuat sekecil mungkin dan tidak perlu
  terbuka ke internet.
- **Beacon node bisa diganti.** Validator client bisa diarahkan ke beacon node lain (atau
  cadangan) tanpa memindah kunci.
- **Satu VC, banyak validator.** Operator besar menjalankan ribuan kunci dari sedikit
  validator client.

## Variasi lokasi

| Setup | Validator client jalan di | Kunci tanda tangan ada di |
| --- | --- | --- |
| Solo staker rumahan | mesin yang sama dengan execution + consensus client | file keystore di mesin itu |
| VC terpisah | mesin lain, terhubung ke beacon node lewat jaringan | mesin VC |
| Remote signer (mis. Web3Signer) | mesin mana pun; VC tidak memegang kunci | layanan signer terpisah; VC minta tanda tangan ke sana |
| DVT (mis. Obol, SSV) | beberapa operator, masing-masing menjalankan VC | kunci dipecah; tanda tangan butuh mayoritas operator |
| Staking pool / exchange | infrastruktur operator | operator; pemilik ETH tidak menjalankan apa pun |

Dari luar, lokasi validator client tidak bisa dilihat: Beacon API cuma menunjukkan tanda tangan
dan jadwal, bukan mesinnya. Petunjuk kecil ada di **graffiti** blok: teks bebas yang diisi
operator, dan sering menyebut nama client (mis. `Nimbus/v26.7.0-…`).

Program VC per tim client: Lighthouse `lighthouse vc`, Prysm `validator`,
Lodestar `lodestar validator`. Teku dan Nimbus bisa menjalankan VC di dalam proses beacon node
atau terpisah.

## Aturan paling penting: satu kunci, satu validator client

Kalau kunci yang sama jalan di dua validator client, keduanya bisa menandatangani dua blok
atau dua suara yang bertentangan. Itu **slashing**: sebagian stake dipotong dan validator
dikeluarkan paksa (cara kerjanya di [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md)). Karena itu, saat pindah mesin, VC lama harus dimatikan dulu dan slashing
protection database ikut dipindah.

## Dua kunci yang berbeda

| | Kunci tanda tangan (signing key) | Withdrawal credentials |
| --- | --- | --- |
| Jenis | BLS, public key 48 byte | alamat execution (prefix `0x01`/`0x02`) |
| Ada di | validator client, harus online | dompet biasa, boleh offline |
| Dipakai untuk | menandatangani blok dan attestation | tujuan penarikan stake dan reward |
| Kalau dicuri | pencuri bisa membuat validator kena slashing | pencuri bisa mengambil dana |

Jadi validator client tidak pernah memegang kunci yang bisa memindahkan ETH. Data validator di
Beacon API memperlihatkan keduanya: public key BLS 48 byte, dan `withdrawal_credentials` berupa
`0x01` + alamat. Alamat penarikan itu akun execution biasa (EOA, code kosong), terpisah dari
kunci tanda tangan. Tanda tangan BLS di blok berukuran 96 byte.

## Coba sendiri

1. Ambil jadwal pembuat blok satu epoch dari Beacon API (endpoint yang juga dipanggil validator
   client).
2. Tunggu salah satu slot di jadwal itu, lalu cocokkan `proposer_index` di beacon block dengan
   jadwalnya. Lihat juga `signature` dan `graffiti`.
3. Buka data salah satu validator: public key, `withdrawal_credentials`, status, saldo.
4. Cek alamat penarikannya lewat JSON-RPC: code, nonce, saldo.

```bash
BEACON=https://ethereum-beacon-api.publicnode.com
curl -s $BEACON/eth/v1/validator/duties/proposer/<epoch>
curl -s $BEACON/eth/v2/beacon/blocks/<slot> | jq '.data.message | {proposer_index}'
curl -s $BEACON/eth/v1/beacon/states/head/validators/<index>
cast code <alamat-penarikan> --rpc-url https://ethereum-rpc.publicnode.com
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md) · [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) →
