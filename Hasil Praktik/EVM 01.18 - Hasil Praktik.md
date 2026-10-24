---
title: EVM 01.18 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.18 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.18 - Finality](../EVM%2001%20-%20Detail/EVM%2001.18%20-%20Finality.md). Dijalankan 2026-10-10 dengan data dari
Beacon API dan JSON-RPC publicnode (Ethereum), dan `bsc-dataseed.bnbchain.org` (BSC).

## Hasil

| Cek | Hasil |
| --- | --- |
| `finality_checkpoints` saat head di epoch 481.228 (slot ke-1 epoch itu) | current justified = epoch **481.227**, finalized = epoch **481.226** |
| Root checkpoint vs root blok | justified 481.227 = `0xbb40…` = blok slot 15.399.264; finalized 481.226 = `0x92e3…` = blok slot 15.399.232 → keduanya blok pertama epoch-nya |
| Satu epoch sebelumnya (state slot 15.399.265, epoch 481.227) | justified 481.226, finalized 481.225 → keduanya maju tepat 1 setelah melewati batas epoch |
| Blok execution di dalam checkpoint finalized | 26.160.329 (`0xd165…`) = **sama persis** dengan `eth_getBlockByNumber("finalized")` |
| Blok execution di dalam checkpoint justified | 26.160.361 = **sama persis** dengan `eth_getBlockByNumber("safe")` |
| `latest` saat itu | 26.160.394 → `finalized` tertinggal **65 blok** (±13 menit) |
| BSC: `finalized` / `safe` / `latest` | 126.784.037 / 126.784.039 / 126.784.041 → `finalized` tertinggal **4 blok** (±1,8 detik dengan blok 0,45 detik) |
| State lebih lama dari ±1 epoch di Beacon API publik | HTTP 403, jadi riwayat checkpoint lebih panjang tidak bisa dibaca |

## Yang tidak dicek

- Rentang waktu finality (±12,8 sampai ±19 menit) diturunkan dari aturan Casper FFG. Yang
  diukur langsung cuma posisi checkpoint dan selisih 65 blok di atas.
- Kebiasaan exchange dan bridge menunggu `finalized`: pengetahuan umum.
- *Inactivity leak* saat finality macet: pengetahuan umum.
- Kehilangan finality di Ethereum mainnet Mei 2023: pengetahuan umum, tidak dicek ulang.
- Mekanisme *fast finality* BSC (BEP-126): pengetahuan umum. Yang diukur cuma jarak
  `finalized` ke `latest`.
