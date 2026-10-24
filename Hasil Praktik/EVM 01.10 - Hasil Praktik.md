---
title: EVM 01.10 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.10 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.10 - Format Alamat 0x](../EVM%2001%20-%20Detail/EVM%2001.10%20-%20Format%20Alamat%200x.md). Dijalankan 2026-10-03 dengan
key default anvil #0.

## Alamat dari public key

Rumus private key → public key → `keccak256` → 20 byte terakhir dicek dengan key default anvil
#0. Hasilnya cocok: 20 byte terakhir hash sama dengan alamat akun #0.

```
public key   = 0x8318535b…0d2aa5 (64 byte)
keccak256    = 0xc1ffd3cfee2d9e5cd67643f8 f39fd6e51aad88f6f4ce6ab8827279cfffb92266
alamat       =                            0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```
