---
title: EVM 02 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 02 - Hasil Praktik

Hasil percobaan untuk materi [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md). Dijalankan 2026-09-24 di `anvil` lokal
(Foundry 1.7.1, chain ID 31337) dan BSC mainnet. Project: `~/Documents/riset/evm/account-model`
(template `forge init`, kontrak `Counter`).

## EOA di anvil

- Akun #0 `anvil` (`0xf39F…2266`): balance 10.000 ETH, nonce 0, code `0x`, storage kosong.
- Alamat acak yang belum pernah dipakai langsung menerima 1 ETH.
- Setelah transfer: nonce pengirim 0 → 1, nonce penerima tetap 0.
- Biaya transfer ETH polos: 0,000021000000021 ETH = 21.000 × 1,000000001 gwei.
- Transaksi dengan calldata ke EOA: sukses, nonce dan gas tetap terpakai.

## Kontrak Counter

`cast compute-address 0xf39F…2266 --nonce 1` → `0xe7f1…0512`, dan `forge create` men-deploy ke
alamat yang persis sama.

| | |
| --- | --- |
| code size | 481 byte |
| nonce | 1 |
| balance | 0 |
| storage slot 0 | `number` (awal 0) |

`setNumber(42)` → `cast storage <kontrak> 0` = `0x…002a` = 42. Setelah `increment()` → 43, sama
dengan `number()` lewat ABI.

Selector `number()` `0x8381f58a`, `setNumber(uint256)` `0x3fb5c1cb`, dan `increment()`
`0xd09de08a` semuanya ditemukan di bytecode.

## Akun di BSC mainnet

Akun-akun dari CAKE Diperiksa Ulang 2026-09-24 dan BNB Diperiksa Ulang 2026-09-24:

| Akun | code | nonce | Artinya |
| --- | ---: | ---: | --- |
| CAKE token `0x0E09…cE82` | 7.285 byte | 1 | kontrak |
| MasterChefV2 `0xa5f8…7652` | 15.151 byte | 1 | kontrak |
| Safe burn `0xceba…4a0e` | 171 byte | 1 | proxy — slot 0 = `0x3e5c63644e683549055b9be8653de26e0b4cd36e` (Gnosis Safe 1.3.0) |
| `0x…dEaD` | 0 | 0 | EOA yang tidak pernah mengirim transaksi; memegang 16,5 jt BNB |
| Bot `0xd099…48d4` | 0 | 6.725 | EOA yang memicu mint-burn mingguan CAKE |

Bot `0xd099…` memanggil Safe → MultiSend → Timelock → MasterChefV2 seminggu sekali (lihat
[EVM 04 - ABI](../EVM%2004%20-%20ABI.md)). Awalnya dikira harian; dikoreksi di Sesi 4.

## EIP-7702 di anvil

Akun #1 (`0x7099…79C8`) mendelegasikan code-nya ke `Counter` sambil memanggil `setNumber(7)`
dalam satu transaksi.

| | Sebelum | Sesudah |
| --- | --- | --- |
| Tipe transaksi | — | `0x4` |
| code akun #1 | `0x` | `0xef0100e7f1725e7734ce288f8367e1bb143e90bb3f0512` (23 byte) |
| `number()` di akun #1 | — | 7 |
| `number()` di Counter asli | 43 | 43 (tidak berubah) |

Percobaan pertama (delegasi tanpa calldata) revert saat estimasi gas. Berhasil setelah delegasi
digabung dengan panggilan `setNumber`.
