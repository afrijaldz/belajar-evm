---
title: EVM 11 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 11 - Hasil Praktik

Hasil percobaan untuk materi [EVM 11 - Events dan Indexing](../EVM%2011%20-%20Events%20dan%20Indexing.md). Dijalankan 2026-09-24. Indexer mini
ditulis sendiri, diuji di `anvil`, lalu dicoba ke RPC Ethereum publik.

Project: `~/Documents/riset/evm/indexer` — `indexer.py` (rebuild saldo dari log, fetch adaptif),
`make_activity.py` (60 transfer acak). Kontrak biaya: `solidity-basics/src/EventCost.sol`.

## Indexer mini

Hasil pada 70 log (1 mint + 69 transfer): **6 holder, semua cocok dengan `balanceOf`**, jumlah
saldo = jumlah mint.

### Temuan tak sengaja: alamat nyasar

Alamat anvil #3 yang ditulis dari ingatan salah (`0x90F7…dE33`, yang benar `0x90F7…b906`). Indexer
menemukannya: **113.370 token (11,3% supply) di alamat tanpa pemilik** — terbakar tanpa sengaja.
Private key akun #4 dari ingatan juga salah. Perbaikan: turunkan dari mnemonic dengan
`cast wallet private-key --mnemonic … --mnemonic-index <n>`.

## Filter topic (data lokal, 70 log)

| Filter | Arti | Log |
| --- | --- | ---: |
| `[T]` | semua Transfer | 70 |
| `[T, #0]` | dari #0 | 20 |
| `[T, null, #0]` | ke #0 | 13 |
| `[T, #0, #1]` | dari #0 ke #1 | 7 |
| `[T, null, [#0, #1]]` | ke #0 **atau** #1 | 30 |
| `[T, 0x0]` | **mint** | 1 |
| `[T, null, alamat nyasar]` | ke alamat nyasar | 3 |

Pola riset: `[T, null, 0xdead]` untuk burn CAKE/UNI, `[T, 0x0]` untuk mint CAKE.

## Batas `eth_getLogs` di RPC publik (Ethereum, UNI Transfer, 2026-09-24)

| RPC | Batas | Cara menolak |
| --- | --- | --- |
| `ethereum-rpc.publicnode.com` | 3.001 blok / 7.701 log lolos; 10.000 blok gagal — kemungkinan batas **jumlah log** | HTTP **403** |
| `eth.drpc.org` | < 1.000 blok | HTTP **400** tanpa pesan |
| `1rpc.io/eth` | **50 blok** | pesan jelas |
| `rpc.flashbots.net` | head tertinggal walau sudah −5 blok | `beyond current head` |
| nodereal BSC (riset CAKE) | 40.000 blok | — |

Jebakan yang ditemui: HTTP 403 untuk User-Agent `Python-urllib` (Cloudflare); node load balancer
yang tertinggal dari `eth_blockNumber`; penolakan rentang yang tidak seragam (403, 400, JSON error).

### Pemecah rentang adaptif

publicnode: **5.000 blok → 12.489 log, 2 request, 22,5 detik** (2.000 lalu 3.001 blok).

UNI ≈ 2,4 Transfer/blok → setahun ≈ 6 jt log ≈ **~3 jam** dari RPC publik. Itu sebabnya riset
memakai Blockscout.

## Reorg

`anvil_reorg(3, …)` mengganti 3 blok terakhir:

| | Sebelum | Sesudah |
| --- | --- | --- |
| Hash blok 68–70 | `0x061a…`, `0xdaa0…`, `0x2acc…` | **semua berubah** |
| Log Transfer di blok itu | 3 | **0** |
| Holder yang saldonya salah di index lama | — | **3** (contoh #1: index 121.451, sebenarnya 222.260) |

(Transaksi pengganti yang disisipkan tidak ikut masuk — kemungkinan nonce. Efek utamanya sama.)

Riset tokenomics aman dari reorg karena membaca data historis, jauh dari ujung chain.

## Event vs storage

Mencatat (from, to, value), transaksi nyata di anvil:

| Cara | Biaya pencatatan |
| --- | ---: |
| Storage, 3 slot (pertama / berikutnya) | 88.561 / 71.461 |
| **Event** `Recorded` (3 topic + 32 byte data) | **1.760** |

Rumus LOG memberi 375 + 1.125 + 256 = **1.756** ✓.
