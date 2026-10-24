---
title: EVM 01.08 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.08 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.08 - Gas](../EVM%2001%20-%20Detail/EVM%2001.08%20-%20Gas.md). Dijalankan 2026-10-03 di `anvil` lokal.

## Loop tanpa akhir berhenti karena gas

Loop `0x5b600056` (`JUMPDEST · PUSH1 0 · JUMP`), gas limit 21.100 (sisa 100 untuk eksekusi):
`cast run -t` menunjukkan gas turun 100 → 99 → 96 → 88 → 87 → … (12 gas per putaran), berhenti
setelah 27 langkah dengan `OutOfGas`.

## Gagal tetap bayar

Loop tanpa akhir dengan limit 50.000 → status 0, gas used **50.000** (seluruh limit), saldo
pengirim turun 44.296.250.000 wei = 50.000 × 885.925 wei, persis.

## Sukses bayar yang terpakai saja

Loop 3 → 0 dengan limit 50.000 → status 1, gas used **21.081** (21.000 + 81), saldo turun
16.349.453.874 wei = 21.081 × 775.554, persis. Sisa 28.919 gas tidak dibayar.
