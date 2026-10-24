---
title: EVM 01.13 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.13 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.13 - EVM, Node, dan Validator](../EVM%2001%20-%20Detail/EVM%2001.13%20-%20EVM%2C%20Node%2C%20dan%20Validator.md). Dijalankan 2026-10-10 ke
RPC publik `ethereum-rpc.publicnode.com` (Ethereum) dan `bsc-dataseed.bnbchain.org` (BSC).

## Hasil

| Cek | Hasil |
| --- | --- |
| `cast client` ke `ethereum-rpc.publicnode.com` | `reth/v2.7.0-3d592ec/x86_64-unknown-linux-gnu` → node RPC ini adalah execution client reth |
| `cast client` ke `bsc-dataseed.bnbchain.org` | `Geth/v1.7.7/linux-amd64/go1.25.12` → client BSC adalah fork geth dengan nomor versi sendiri |
| Field `miner` 20 blok Ethereum (26.160.207–26.160.226) | 4 alamat (12, 6, 1, dan 1 blok); salah satunya `0x0000…0000` |
| Field `miner` 20 blok BSC (126.779.522–126.779.541) | hanya 3 alamat (8, 6, dan 6 blok) |
| `getValidators()` di kontrak sistem BSC `0x…1000` | 45 alamat; ketiga pembuat blok di atas ada di daftar; code kontrak 29.871 byte |
| Blok Ethereum 26.160.253 lewat Beacon API | dibuat validator nomor 2.092.386 (stake 32 ETH), sementara `miner`-nya cuma `0x3963…aa49` (lihat [EVM 01.14 - Hasil Praktik](EVM%2001.14%20-%20Hasil%20Praktik.md)) |

## Yang tidak dicek

- Urutan BSC "satu validator membuat beberapa blok berturut-turut per giliran" berasal dari
  pengetahuan umum. Yang dicek cuma jumlah blok per alamat, bukan urutannya.
