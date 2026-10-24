---
title: EVM 01.05 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.05 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.05 - Environment](../EVM%2001%20-%20Detail/EVM%2001.05%20-%20Environment.md). Dijalankan 2026-10-03 di `anvil`
lokal.

## CALLER vs ORIGIN

Kontrak A menyimpan `CALLER`, `CALLVALUE`, `ORIGIN`, `NUMBER`, `CHAINID` ke storage.

| | Dipanggil langsung dari wallet + 7 wei | Dipanggil lewat kontrak B |
| --- | --- | --- |
| `CALLER` | `0xf39F…2266` (wallet) | `0x84ea…7feb` (**B**) |
| `CALLVALUE` | 7 | 0 |
| `ORIGIN` | `0xf39F…2266` | `0xf39F…2266` (tetap wallet) |
| `NUMBER` | 34 | 35 |
| `CHAINID` | `0x7a69` = 31337 | 31337 |

## Jebakan yang ditemui

`cast send` ke B tanpa `--gas-limit` memperkirakan gas terlalu kecil. Call ke A gagal
`OutOfGas`, tapi transaksinya tetap **sukses**, karena opcode `CALL` tidak meneruskan kegagalan.
