---
title: EVM 01.15 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.15 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.15 - Validator Client](../EVM%2001%20-%20Detail/EVM%2001.15%20-%20Validator%20Client.md). Dijalankan 2026-10-10 dengan data
dari Beacon API dan JSON-RPC publicnode.

## Hasil

| Cek | Hasil |
| --- | --- |
| `/eth/v1/validator/duties/proposer/481224` (endpoint yang juga dipanggil validator client) | 32 entri = satu pembuat blok per slot, 32 validator berbeda; slot pertama 15.399.168 = 481.224 × 32 |
| Jadwal slot 15.399.172 | validator **2.365.600** |
| Beacon block head slot 15.399.172 | `proposer_index` **2.365.600** → sesuai jadwal; signature 96 byte (BLS); graffiti `Nimbus/v26.7.0-4110bc-stateofus` (graffiti diisi bebas oleh operator, di sini menunjukkan client Nimbus) |
| Validator 2.092.386 (dari [EVM 01.14 - Hasil Praktik](EVM%2001.14%20-%20Hasil%20Praktik.md)) | public key **48 byte** (BLS); `withdrawal_credentials` `0x01` + alamat `0x84af…9985`; aktif sejak epoch 396.373; saldo 32,0636 ETH; `slashed` false |
| Alamat penarikan `0x84af…9985` lewat JSON-RPC | code 0 byte (EOA), nonce 32, saldo 8,57 ETH → kunci penarikan adalah akun execution biasa, terpisah dari kunci tanda tangan validator |

Langkah 2 di materi (satu attestation per epoch) dicek lewat data komite di
[EVM 01.17 - Hasil Praktik](EVM%2001.17%20-%20Hasil%20Praktik.md).

## Yang tidak dicek

Bagian materi berikut berasal dari pengetahuan umum:

- Langkah 4: slashing protection database.
- Tabel "Variasi lokasi" (solo staker, VC terpisah, remote signer, DVT, staking pool).
- Nama program VC per tim client.
- Kolom "Kalau dicuri" di tabel "Dua kunci yang berbeda".
