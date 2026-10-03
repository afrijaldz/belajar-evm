---
title: EVM 23 - Multi-Chain
tags: [learning, evm, multichain, create2, bridge, l2]
updated: 2026-09-24
---

# Multi-Chain

Catatan dari Sesi 23 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Empat mainnet dibaca langsung (Ethereum, BSC, Base,
Arbitrum), dua di-fork (Base, BSC) untuk deploy; wallet testnet Sesi 21 masih 0, jadi tidak ada deploy
sungguhan. Lanjutan dari [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md).

Script: `~/Documents/riset/evm/multichain/` — `chains.py`, `bridge.py` (langsung ke mainnet), `multichain.py`
(butuh fork Base di port 8550 dan BSC di 8551, jalankan dari folder `vault-project`).

## Empat chain berdampingan

| Chain | ID | Block time | Multicall3 `0xcA11…CA11` | CREATE2 deployer `0x4e59…956C` | `eth_blockNumber` | `block.number` dilihat kontrak |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Ethereum | 1 | 12,18 s | 3.808 byte | 69 byte | 26.046.755 | 26.046.755 |
| BSC | 56 | 0,45 s | 3.808 byte | 69 byte | 123.741.944 | 123.741.956 |
| Base | 8453 | 2,00 s | 3.808 byte | 69 byte | 51.728.400 | 51.728.401 |
| Arbitrum | 42161 | 0,27 s | 3.808 byte | 69 byte | **508.415.943** | **26.046.756** |

- Multicall3 dan deployer CREATE2 Arachnid ada **di alamat sama** di keempat chain — fondasi deploy
  multi-chain.
- **Di Arbitrum, `block.number` di dalam kontrak = nomor blok Ethereum (L1)**, bukan L2. `block.number`
  dibaca lewat `Multicall3.getBlockNumber()`.

### Bahaya `block.number` untuk waktu

Kunci "100.000 blok": BSC ≈ **12,5 jam**, Ethereum dan Arbitrum ≈ **14 hari** — kode identik. Kontrak
multi-chain memakai `block.timestamp` untuk mengukur waktu. (Riset CAKE juga pernah salah hitung block time
BSC — Lapisan Kedua BNB dan CAKE.)

## CREATE vs CREATE2

`Vault` di-deploy dari wallet baru yang sama di fork Base dan fork BSC. Di BSC ada satu transaksi ekstra
lebih dulu, jadi nonce berbeda.

| | Base (8453) | BSC (56) |
| --- | --- | --- |
| **CREATE** (nonce 0 vs 1) | `0x750f69147b30B7A8520e36B8d4098447Fa88CA05` | `0x2e197f4a353feF322B5F3e785C0872Bf914ce0aF` — **beda** |
| **CREATE2** lewat Arachnid, salt `0x…17` | `0x94A2e88Da164797087B2d39737bb187B24725dAB` | `0x94A2e88Da164797087B2d39737bb187B24725dAB` — **sama**, 2.084 byte |

```
alamat CREATE2 = keccak256(0xff ‖ deployer ‖ salt ‖ keccak256(initCode))[12:]
cast create2 --deployer 0x4e59b44847b379578588920cA78FbF26c0B4956C --salt <salt> --init-code <bytecode>
```

Alamat CREATE2 dihitung **sebelum** menyentuh chain apa pun. Syarat: bytecode identik byte per byte —
termasuk metadata; `optimizer_runs` berbeda → alamat berbeda ([EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md)).
Deployer Arachnid menerima calldata = `salt ‖ initCode`.

## Replay antar chain

Transaksi ditandatangani untuk chain 8453 → **diterima** di fork Base (status `0x1`), **ditolak** di fork BSC:
`invalid chain id for signer`. Chain ID ikut ditandatangani (EIP-155, [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)).

## Bridge: UNI di L2 (lock-and-mint)

UNI asli dikunci di kontrak bridge L1; versi L2 dicetak. Invariant bridge sehat:
**UNI terkunci di L1 ≥ supply UNI di L2**.

| L2 | UNI L2 | Fungsi asal-token | Supply L2 | Terkunci di L1 (escrow) | Kelebihan |
| --- | --- | --- | ---: | ---: | ---: |
| Base | `0xc3De830EA07524a0761646a6a4e4be0e114a3C83` | `remoteToken()` | 157.825 | 236.720 (`0x3154…2c35`) | **+78.895** |
| Optimism | `0x6fd9d7AD17242c41f7131d257212c54A0e816691` | `l1Token()` | 73.573 | 82.497 (`0x99c9…4be1`) | +8.924 |
| Arbitrum | `0xFa7F8980b0f1E64A2062791cc3b0871572f1F7f0` | `l1Address()` | 998.081 | 1.038.118 (`0xa3a7…0eec`) | +40.037 |

Asal token = UNI L1 di ketiganya (dibaca on-chain). Kelebihan escrow kemungkinan besar **burn yang sedang
dalam perjalanan**: burn L2 langsung mengurangi supply L2, escrow L1 baru dilepas ke `0xdead` setelah
withdrawal final (~7 hari). Menyambung [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md): ~128.000 UNI kemungkinan akan tiba di
`0xdead` beberapa hari ke depan. Sebagian kelebihan bisa juga UNI yang terkirim langsung ke bridge — tidak
bisa dipisahkan dari data ini.

## Checklist kontrak multi-chain

| Hal | Kenapa |
| --- | --- |
| Waktu pakai `block.timestamp`, bukan `block.number` | block time 0,27–12 s, Arbitrum `block.number` = L1 |
| Deploy via CREATE2 dengan salt + bytecode sama | alamat sama di semua chain |
| Kunci pengaturan compiler | beda bytecode = beda alamat CREATE2 + verifikasi gagal |
| Jangan asumsikan token sama = alamat sama | UNI punya alamat berbeda di tiap L2 |
| Cek `evm_version` yang didukung chain target | opcode baru belum tentu ada di semua chain |
| Pantau invariant bridge (escrow ≥ supply L2) | bridge adalah titik kepercayaan terbesar |
