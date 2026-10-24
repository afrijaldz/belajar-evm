---
title: EVM 22 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 22 - Hasil Praktik

Hasil percobaan untuk materi [EVM 22 - Monitoring](../EVM%2022%20-%20Monitoring.md). Dijalankan 2026-09-24 di `anvil` (eksploit dan reorg
simulasi) dan Ethereum mainnet (laju burn UNI). Kode: `~/Documents/riset/evm/monitor/` — `monitor.py`
(Vault), `uni_burn_rate.py` (mainnet). Saat itu Sesi 21 masih menunggu faucet.

## Monitor `Vault` di anvil

| Kejadian | Output monitor |
| --- | --- |
| Deposit / withdraw normal | tercatat per blok |
| Withdraw 120 token | `ALERT large withdrawal 120.0000 by 0x7099… in block 8` |
| Saldo token vault dipotong lewat `anvil_setStorageAt` (simulasi eksploit) | `ALERT INSOLVENT asset 0xe7f1… holds 30.0000 but owes 70.0000` |
| `anvil_reorg` 2 blok, diganti deposit 3 ETH akun #5 | `ALERT reorg at block 10`, `… block 9`, lalu blok 9 baru dibaca |

Setelah reorg, alert INSOLVENT hilang: manipulasi storage ada di blok yang dibatalkan.

Kelemahan yang terlihat: deposit di blok 10 sudah terlanjur dilaporkan sebelum blok itu dibatalkan; alert
INSOLVENT berulang tiap polling; output cuma `print`.

## Jebakan yang ditemui: hash dari ingatan

Topic `Deposited` dan selector `totalDeposits` yang ditulis dari ingatan salah (`0x2da4…`, `0xccd6e1e5`;
benar `0x8752…68a7`, `0xe9403256`). Monitor diam tanpa error. Semua hash kemudian diambil dari `cast keccak` /
`cast sig`. Pola kesalahan sama dengan alamat dan key di Sesi 11.

## Mainnet: laju burn UNI

`uni_burn_rate.py`: hitung event `Released` Firepit mainnet di N blok terakhir.

| Jendela | Released | UNI |
| --- | ---: | ---: |
| 1.800 blok (6,07 jam) | **1** (nonce 1.360) | 4.000 → **659 UNI/jam** |

Tidak cocok dengan +68.000 UNI "dalam beberapa jam" di [EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md). Semua
transfer UNI → `0xdead` sejak 2026-09-23 06:00 UTC (Blockscout):

| Pengirim | Kontrak | Asal | UNI |
| --- | --- | --- | ---: |
| `0x85001cc4867c5e1c22da4b79bb8852b9e2a06da0` | `L1ERC20Gateway` (proxy) | gateway gaya Arbitrum, chain belum diidentifikasi | 60.000 (30 tx) |
| `0x3154cf16ccdb4c6d922629664174b904d80f2c35` | `L1StandardBridge` | **Base** | 20.000 |
| `0xc6208ce00b3f88c6555833ab9bbc943717dfd941` | kontrak tanpa nama | belum diidentifikasi | 12.000 |
| `0xa3a7b6f88361f48403514059f1f16c8e78d60eec` | `L1ERC20Gateway` | **Arbitrum** | 8.000 |
| `0x6569925aac77d6b8bb085f31f9828ff80d5a0c44` | `NttManager` | Wormhole NTT | 6.000 |
| `0x9b715efb4ebdb14862d079a841ba4983ffd69236` | bot Sesi 6 | Firepit **Ethereum** | 4.000 |
| `0x99c9fc46f92e8a1c0dec1b1747d010903e884be1` | `L1StandardBridge` | **Optimism** | 2.000 |
| **Total** | 51 transfer | | **112.000** |

Paket 2.000 UNI di atas sesuai threshold Firepit L2. Firepit mainnet cuma **~4%** burn dalam sampel ini, jadi
monitor yang cuma melihat `Released` di mainnet melewatkan ~96% burn.

Koreksi sudah ditambahkan ke [EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md) dan
UNI Diperiksa dengan Saringan Lima Chain.
