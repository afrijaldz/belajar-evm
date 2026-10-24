---
title: EVM 02.02 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 02.02 - Hasil Praktik

Hasil percobaan untuk materi [EVM 02.02 - Nonce](../EVM%2002%20-%20Detail/EVM%2002.02%20-%20Nonce.md). Dijalankan 2026-10-03 di `anvil` lokal dan
Ethereum mainnet.

## Aturan nonce di anvil

| Percobaan | Hasil |
| --- | --- |
| Akun #0 kirim ETH 3 kali ke akun #1 | nonce pengirim 59 → 62, nonce penerima tetap **0** |
| Kirim lagi dengan `--nonce 1` (sudah terpakai) | `error code -32003: nonce too low` |
| Raw tx yang sama di-`publish` dua kali | pertama sukses, kedua `nonce too low` (replay ditolak) |
| Deploy init code yang langsung `REVERT` (`0x60006000fd`) | status `0x0`, gas tetap terpakai, nonce tetap naik 65 → 66 |
| Kirim tx dengan nonce 64 saat nonce akun 63 | tx tidak masuk blok; `eth_getTransactionCount … pending` tetap `0x3f` (63) |
| Lalu kirim tx nonce 63 | keduanya masuk blok, nonce jadi **65** |
| Automine dimatikan, kirim nonce 69 (2 gwei), lalu nonce 69 lagi (3 gwei) | tx kedua **menggantikan** tx pertama; blok `0x45` hanya berisi tx kedua (`value` 2) |
| Kirim nonce 69 ketiga kalinya dengan fee yang sama (3 gwei) | `replacement transaction underpriced` |

## Nonce kontrak

Factory dari bytecode mentah `600060006000f000` (`CREATE(0,0,0)` lalu `STOP`):

| Langkah | Hasil |
| --- | --- |
| Factory baru di-deploy | nonce factory = **1**, bukan 0 (EIP-161) |
| Factory dipanggil 2 kali (2× `CREATE`) | nonce factory = **3** |
| `cast compute-address <factory> --nonce 1` dan `--nonce 2` | kedua alamat itu sekarang ada, masing-masing dengan nonce **1** |

## Data mainnet Ethereum

| Akun | nonce | Artinya |
| --- | --- | --- |
| Vitalik `0xd8dA…6045` | **5966** | sudah mengirim 5.966 transaksi |
| USDC `0xA0b8…eB48` | **1** | kontrak yang belum pernah membuat kontrak lain |
| `0x…dEaD` | **0** | tidak pernah mengirim apa pun; private key-nya tidak diketahui siapa pun |
