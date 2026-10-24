---
title: EVM 01.03 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.03 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.03 - Stack](../EVM%2001%20-%20Detail/EVM%2001.03%20-%20Stack.md). Dijalankan 2026-10-03 di `anvil` lokal.

## Overflow dan padding alamat

Bytecode `PUSH32 0xff…ff (max) · PUSH1 2 · ADD · PUSH1 0 · SSTORE · CALLER · PUSH1 1 · SSTORE`:

- slot 0 = `0x…01` (max + 2 memutar jadi 1, tanpa revert)
- slot 1 = `0x000000000000000000000000f39fd6e5…92266` (alamat pemanggil dipad ke 32 byte)
