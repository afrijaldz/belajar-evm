---
title: EVM 23 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 23 - Hasil Praktik

Hasil percobaan untuk materi [EVM 23 - Multi-Chain](../EVM%2023%20-%20Multi-Chain.md). Dijalankan 2026-09-24: empat mainnet dibaca langsung
(Ethereum, BSC, Base, Arbitrum), dua di-fork (Base, BSC) untuk deploy. Wallet testnet Sesi 21 masih 0, jadi
tidak ada deploy sungguhan. Script: `~/Documents/riset/evm/multichain/` — `chains.py`, `bridge.py` (langsung ke
mainnet), `multichain.py` (butuh fork Base di port 8550 dan BSC di 8551, jalankan dari folder `vault-project`).

## Empat chain berdampingan

| Chain | ID | Block time | Multicall3 `0xcA11…CA11` | CREATE2 deployer `0x4e59…956C` | `eth_blockNumber` | `block.number` dilihat kontrak |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Ethereum | 1 | 12,18 s | 3.808 byte | 69 byte | 26.046.755 | 26.046.755 |
| BSC | 56 | 0,45 s | 3.808 byte | 69 byte | 123.741.944 | 123.741.956 |
| Base | 8453 | 2,00 s | 3.808 byte | 69 byte | 51.728.400 | 51.728.401 |
| Arbitrum | 42161 | 0,27 s | 3.808 byte | 69 byte | **508.415.943** | **26.046.756** |

`block.number` dibaca lewat `Multicall3.getBlockNumber()`. Riset CAKE juga pernah salah hitung block time
BSC — Lapisan Kedua BNB dan CAKE.

## CREATE vs CREATE2

`Vault` di-deploy dari wallet baru yang sama di fork Base dan fork BSC. Di BSC ada satu transaksi ekstra
lebih dulu, jadi nonce berbeda.

| | Base (8453) | BSC (56) |
| --- | --- | --- |
| **CREATE** (nonce 0 vs 1) | `0x750f69147b30B7A8520e36B8d4098447Fa88CA05` | `0x2e197f4a353feF322B5F3e785C0872Bf914ce0aF` — **beda** |
| **CREATE2** lewat Arachnid, salt `0x…17` | `0x94A2e88Da164797087B2d39737bb187B24725dAB` | `0x94A2e88Da164797087B2d39737bb187B24725dAB` — **sama**, 2.084 byte |

## Replay antar chain

Transaksi ditandatangani untuk chain 8453 → **diterima** di fork Base (status `0x1`), **ditolak** di fork BSC:
`invalid chain id for signer`.

## Bridge UNI

| L2 | UNI L2 | Fungsi asal-token | Supply L2 | Terkunci di L1 (escrow) | Kelebihan |
| --- | --- | --- | ---: | ---: | ---: |
| Base | `0xc3De830EA07524a0761646a6a4e4be0e114a3C83` | `remoteToken()` | 157.825 | 236.720 (`0x3154…2c35`) | **+78.895** |
| Optimism | `0x6fd9d7AD17242c41f7131d257212c54A0e816691` | `l1Token()` | 73.573 | 82.497 (`0x99c9…4be1`) | +8.924 |
| Arbitrum | `0xFa7F8980b0f1E64A2062791cc3b0871572f1F7f0` | `l1Address()` | 998.081 | 1.038.118 (`0xa3a7…0eec`) | +40.037 |

Asal token = UNI L1 di ketiganya (dibaca on-chain). Menyambung [EVM 22 - Monitoring](../EVM%2022%20-%20Monitoring.md): ~128.000 UNI
kemungkinan akan tiba di `0xdead` beberapa hari ke depan.
