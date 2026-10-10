---
title: EVM 01.15 - Validator Client
tags: [learning, evm, ethereum]
updated: 2026-10-10
---

# Validator Client

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md) · [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) →

Pertanyaan: validator client itu jalan di mana?

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
   dianggap benar).
3. Kalau dapat giliran membuat blok: minta beacon node menyiapkan blok (beacon node meminta
   isinya ke execution client, lihat [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md)),
   tanda tangani, lalu kirim kembali ke beacon node untuk disebar.
4. Menyimpan **slashing protection database**: catatan semua yang sudah ditandatangani, supaya
   tidak pernah menandatangani dua hal yang bertentangan.

Langkah 2 dan 4 dari pengetahuan umum; yang dicek langsung cuma jadwal pembuat blok (lihat
Bukti).

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

Tabel ini dari pengetahuan umum. Dari luar, lokasi validator client tidak bisa dilihat: Beacon
API cuma menunjukkan tanda tangan dan jadwal, bukan mesinnya.

Program VC per tim client (pengetahuan umum): Lighthouse `lighthouse vc`, Prysm `validator`,
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

Jadi validator client tidak pernah memegang kunci yang bisa memindahkan ETH. Kolom "kalau
dicuri" dari pengetahuan umum.

## Bukti (2026-10-10)

Data dari Beacon API dan JSON-RPC publicnode.

| Cek | Hasil |
| --- | --- |
| `/eth/v1/validator/duties/proposer/481224` (endpoint yang juga dipanggil validator client) | 32 entri = satu pembuat blok per slot, 32 validator berbeda; slot pertama 15.399.168 = 481.224 × 32 |
| Jadwal slot 15.399.172 | validator **2.365.600** |
| Beacon block head slot 15.399.172 | `proposer_index` **2.365.600** → sesuai jadwal; signature 96 byte (BLS); graffiti `Nimbus/v26.7.0-4110bc-stateofus` (graffiti diisi bebas oleh operator, di sini menunjukkan client Nimbus) |
| Validator 2.092.386 (dari [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md)) | public key **48 byte** (BLS); `withdrawal_credentials` `0x01` + alamat `0x84af…9985`; aktif sejak epoch 396.373; saldo 32,0636 ETH; `slashed` false |
| Alamat penarikan `0x84af…9985` lewat JSON-RPC | code 0 byte (EOA), nonce 32, saldo 8,57 ETH → kunci penarikan adalah akun execution biasa, terpisah dari kunci tanda tangan validator |

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.14 - Execution Client dan Consensus Client](EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md) · [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md) →
