---
title: EVM 02.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 02.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 02.01 - EOA dan Contract Account](../EVM%2002%20-%20Detail/EVM%2002.01%20-%20EOA%20dan%20Contract%20Account.md). Dijalankan 2026-10-03 di
Ethereum mainnet dan `anvil` lokal.

## Akun nyata di Ethereum mainnet

| Akun | code | nonce | Jenis |
| --- | ---: | ---: | --- |
| `0xd8dA…6045` (wallet publik Vitalik Buterin) | **23 byte** `0xef01005a7f…6f6d` | 5.966 | EOA dengan delegasi EIP-7702 ke kontrak `0x5a7f…6f6d` (11.162 byte) |
| `0xA0b8…eB48` (USDC) | 2.186 byte | 1 | contract (proxy) |
| `0xC02a…6Cc2` (WETH) | 3.124 byte | 1 | contract |
| `0x…dEaD` | 0 | 0 | EOA tanpa pemilik yang diketahui |

Diagram "Gambaran" di materi awalnya memakai angka EOA `balance: 5,7 ETH` dan `nonce: 5966`
(nonce yang sama dengan `0xd8dA…6045`).

## Membuat masing-masing di anvil

- **EOA**: `cast wallet new` membuat key + alamat **secara offline**, tanpa transaksi. Alamat
  baru `0x8830…cB91` langsung "ada" di chain dengan nonce 0, code `0x`, saldo 0.
- **Contract**: `cast compute-address 0xf39F…2266 --nonce 58` memprediksi `0xc351…1181`;
  deploy berikutnya dari akun itu mendarat **persis** di alamat tersebut, dengan nonce 1 dan
  code `0x00`.
