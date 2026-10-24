---
title: EVM 01.12 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.12 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.12 - Native Token](../EVM%2001%20-%20Detail/EVM%2001.12%20-%20Native%20Token.md). Dijalankan 2026-10-03 di `anvil`
lokal, Ethereum mainnet, dan BSC mainnet.

## Hasil

| Cek | Hasil |
| --- | --- |
| Transfer 1 ETH ke `0x…bEEF` di anvil | status 1, gas **21.000** (`0x5208`, hanya biaya dasar), **0 log**; saldo 0 → 1 ETH; code penerima `0x` (EOA biasa) |
| BNB di Ethereum (`0xB8c7…DD52`) | ada code 5.355 byte, `symbol()` = `"BNB"`, `totalSupply()` = 392.908,13 BNB → kontrak ERC-20 |
| BNB di BSC, `eth_getBalance(0xdEaD)` | 16.498.950,81 BNB, naik dari 16.498.949,12 beberapa jam sebelumnya (burn BNB jalan terus) |
| WETH mainnet `0xC02a…6Cc2` | saldo ETH native kontrak = `totalSupply()` WETH = **2.027.136,551081447271723759**, sama sampai wei terakhir |

Diagram world state di materi awalnya memakai angka WETH ini: `balance: 2.027.136 ETH` dan
`totalSupply → 2.027.136 WETH`.
