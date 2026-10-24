---
title: EVM 01.06 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.06 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.06 - Storage](../EVM%2001%20-%20Detail/EVM%2001.06%20-%20Storage.md). Dijalankan 2026-10-03: storage UNI
dibaca dengan `cast storage` ke `ethereum-rpc.publicnode.com` (Ethereum mainnet), tabel gas
diukur di `anvil` lokal.

## Storage kontrak UNI `0x1f98…F984`

```text
┌─────────────────────────────────┬──────────────────────────────────────────────────────┐
│ key (slot)                      │ value (32 byte)                                      │
├─────────────────────────────────┼──────────────────────────────────────────────────────┤
│ 0                               │ 0x…033b2e3c9fd0803ce8000000 = 10²⁷ → totalSupply     │
│ 1                               │ 0x…1a9c8182c09f50c8318d769245bea52c32be35bc → minter │
│ 2                               │ 0x…65920080 = 2024-01-01 → mintingAllowedAfter       │
│ 3                               │ 0 (mapping allowances, isinya di slot lain)          │
│ 4                               │ 0 (mapping balances, isinya di slot lain)            │
│ keccak256(pad(0xdEaD) ‖ pad(4)) │ 112.515.581,08 UNI → balances[0xdEaD]                │
│   = 0x42c6…47dd                 │                                                      │
└─────────────────────────────────┴──────────────────────────────────────────────────────┘
```

- Slot 0 sama dengan `totalSupply()` = 10²⁷.
- Nilai di `cast index address 0x…dEaD 4` sama dengan `balanceOf(0xdEaD)` =
  112515581083689919081184389.
- Key yang sama dengan slot 3 dan 5 bernilai 0, jadi `balances` ada di slot 4.

## Gas storage terukur

Kontrak kecil di `anvil`, diukur dengan `cast run -vvvvv`. Angka terukur termasuk
`PUSH`/`CALLDATALOAD`.

| Operasi | Gas (spesifikasi) | Terukur |
| --- | ---: | ---: |
| `SLOAD` cold | 2.100 | 2.103 |
| `SLOAD` warm | 100 | 2.206 (cold + warm) |
| `SSTORE` 0 → bukan 0 | 22.100 | 22.109 |
| `SSTORE` bukan 0 → bukan 0 lain | 5.000 | 5.009 |
| `SSTORE` nilai sama | 2.200 | 2.209 |
| `SSTORE` bukan 0 → 0 | 5.000, refund 4.800 di akhir tx | 5.009 |
