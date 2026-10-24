---
title: EVM 01.04 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.04 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.04 - Memory dan Calldata](../EVM%2001%20-%20Detail/EVM%2001.04%20-%20Memory%20dan%20Calldata.md). Dijalankan 2026-10-03 di
`anvil` lokal.

## Calldata dan memory antar call

Kontrak kecil yang membaca calldata lalu menulis memory, dipanggil dua kali dengan calldata
`transfer(0xdEaD, 1000)`:

| Yang diukur | Hasil call 1 | Hasil call 2 |
| --- | --- | --- |
| `CALLDATASIZE` | 68 (= 4 + 32 + 32) | 68 |
| `CALLDATALOAD(0)` | `0xa9059cbb000…` (selector + 28 byte awal argumen) | sama |
| `MLOAD(0)` sebelum ditulis | 0 | **0**, walaupun call 1 sudah menulis 42 ke sana |
| `MSIZE` setelah satu `MSTORE` | 32 | 32 |

## Biaya membesarkan memory

Satu `MSTORE` di offset makin jauh (gas eksekusi seluruh call):

| Offset | Ukuran memory | Gas |
| ---: | ---: | ---: |
| 0 | 32 byte | 15 |
| 1.024 | ~1 KB | 113 |
| 32.768 | ~32 KB | 5.139 |
| 1.048.576 | ~1 MB | 2.195.599 |

Ukuran naik 32.768x, gas naik ~146.000x, sesuai rumus 3 × word + word² / 512.
