---
title: EVM 01.02 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.02 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.02 - Opcode](../EVM%2001%20-%20Detail/EVM%2001.02%20-%20Opcode.md). Dijalankan 2026-10-03 di Ethereum mainnet
(baca saja) dan `anvil` lokal.

## Disassemble Multicall3

`cast disassemble $(cast code 0xcA11…CA11)` (Multicall3, Ethereum) → **3.808 byte = 2.160
opcode, 78 jenis berbeda**. Awal bytecode-nya sama dengan contoh di materi: `PUSH1 0x80 ·
PUSH1 0x40 · MSTORE`, cek calldata < 4 byte, ambil selector, lalu `PUSH4 0x4d2301cc`
(`getEthBalance(address)`, dicek dengan `cast sig`).

Opcode yang paling sering muncul:

| Opcode | Jumlah |
| --- | ---: |
| `PUSH` | 552 |
| `DUP` | 396 |
| `SWAP` | 190 |
| `JUMPDEST` | 151 |
| `POP` | 143 |

## Byte tak terdefinisi

Byte `0x0c` dijalankan di `anvil` → status 0, gas used 50.000 = seluruh gas limit.
