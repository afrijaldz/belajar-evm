---
title: EVM 01.07 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.07 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.07 - Program Counter](../EVM%2001%20-%20Detail/EVM%2001.07%20-%20Program%20Counter.md). Dijalankan 2026-10-03 di `anvil`
lokal.

## Loop hitung mundur 3 → 0

Code `0x60035b600190038060025700`, jejak dari `cast run <tx> -t`. Tabel jejak di materi diambil
dari run ini. Hasil akhirnya: tiga putaran, `JUMPI` terakhir tidak lompat (syarat 0), `STOP` di
PC 11, **gas eksekusi 81**.

## Lompatan ilegal

Code `0x600456605b00` (`PUSH1 4 · JUMP · PUSH1 0x5b · STOP`) → `InvalidJump`, status transaksi
0, dan seluruh gas limit (**100.000**) habis.
