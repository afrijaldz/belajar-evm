---
title: EVM 07 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 07 - Hasil Praktik

Hasil percobaan untuk materi [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md). Dijalankan 2026-09-24 dengan test
Foundry (Solidity ^0.8.24, forge 1.7.1). Project: `~/Documents/riset/evm/solidity-basics` —
`src/` (Types, Packing, Locations, Vault, Exercise) dan `test/`. Jalankan: `forge test -vv`.

## Storage packing

| Kontrak | Urutan | Slot terpakai | Gas `setAll` pertama |
| --- | --- | ---: | ---: |
| `Unpacked` | `uint128 a; uint256 b; uint128 c;` | 3 | 68.316 |
| `Packed` | `uint128 a; uint128 c; uint256 b;` | 2 | 46.466 |

**Hemat 21.850 gas** hanya dari urutan deklarasi. Slot 0 `Packed` = `…0003` (c, byte 16–31) +
`…0001` (a, byte 0–15).

## storage vs memory vs calldata

| Lokasi | Hasil test |
| --- | --- |
| `memory` | `closeWithMemory` → `active` tetap `true`, perubahan hilang |
| `storage` | `closeWithStorage` → `active = false`, tersimpan |
| `calldata` | **58.362** gas vs 85.794 (`memory`) untuk array 100 elemen, −32% |

Compiler memberi warning *"Function state mutability can be restricted to view"* di
`closeWithMemory`.

## Visibility

`secretCode` (`private`) terbaca dari slot 1 dengan `vm.load`.

## Modifier dan custom error

| | `require(…, "string")` | custom error |
| --- | ---: | ---: |
| Revert data | 100 byte | 36 byte |
| Gas panggilan gagal | 8.152 | **3.668** |

`setCode` punya `onlyOwner whenNotPaused`: saat paused, orang asing dapat `NotOwner`, bukan
`Paused`.

## Menerima ETH

Kode: `src/Payable.sol`, `test/Payable.t.sol`. Hasil detail di [EVM 07.01 - Hasil Praktik](EVM%2007.01%20-%20Hasil%20Praktik.md) dan
[EVM 07.02 - Hasil Praktik](EVM%2007.02%20-%20Hasil%20Praktik.md).

## Latihan

`DepositBank` di `src/Exercise.sol` sudah diverifikasi bisa diselesaikan (test
`test/Exercise.t.sol` bisa dibuat lulus).
