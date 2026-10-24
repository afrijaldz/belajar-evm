---
title: EVM 08 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 08 - Hasil Praktik

Hasil percobaan untuk materi [EVM 08 - ERC-20 dari Nol](../EVM%2008%20-%20ERC-20%20dari%20Nol.md). Dikerjakan 2026-09-24: token ditulis
sendiri tanpa library, diuji dengan Foundry (14 test, 2 di antaranya fuzz × 256 run), lalu
di-deploy ke `anvil`. Project: `~/Documents/riset/evm/solidity-basics` — `src/MyToken.sol`,
`test/MyToken.t.sol`.

## Ukuran bytecode

MyToken **3.091 byte**. Pembanding: CAKE 7.285 byte — sisanya delegasi/checkpoint suara dan
`mint` owner.

## Selector dibandingkan dengan riset

| | MyToken | Dipakai di riset |
| --- | --- | --- |
| `balanceOf(address)` | `0x70a08231` | ✓ `eth_call` CAKE, UNI, ETHFI |
| `totalSupply()` | `0x18160ddd` | ✓ |
| `transfer(address,uint256)` | `0xa9059cbb` | ✓ |
| `Transfer(address,address,uint256)` | `0xddf252ad…` | ✓ filter `eth_getLogs` burn |

Script riset (eth_call selector mentah, `cast logs` topic Transfer) jalan di MyToken tanpa
perubahan. Di Sesi 4, mint CAKE dikenali dari Transfer `0x0 → …`.

## Bug transfer ke diri sendiri

Dengan bug urutan baca saldo, transfer 100 ke diri sendiri → saldo **1.000.000 → 1.000.100**.
Yang menangkap: test manual `test_TransferToSelfKeepsBalance`, bukan fuzz (fuzz memakai
`vm.assume(to != alice)`).

## Gas nyata (anvil, transaksi terpisah)

| Transaksi | Gas |
| --- | ---: |
| transfer ke holder **baru** | 52.349 |
| transfer ke holder **lama** | 35.249 |
| **Selisih** | **17.100** |
| approve (slot baru) | 46.710 |

Pembanding di dalam satu test Foundry: "holder lama" = 5.272 gas, di anvil 35.249.

`transferFrom` dengan allowance terbatas vs unlimited (dalam test, relatif): 33.284 vs 27.981.

## Latihan

`test/ExerciseBurn.t.sol` sudah diverifikasi bisa diselesaikan.
