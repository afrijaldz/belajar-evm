---
title: EVM 01.14 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.14 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.14 - Execution Client dan Consensus Client](../EVM%2001%20-%20Detail/EVM%2001.14%20-%20Execution%20Client%20dan%20Consensus%20Client.md). Dijalankan
2026-10-10 dengan data dari publicnode: Beacon API `ethereum-beacon-api.publicnode.com` dan
JSON-RPC `ethereum-rpc.publicnode.com`.

## Hasil

| Cek | Hasil |
| --- | --- |
| Beacon API `/eth/v1/node/version` | `Lighthouse/v8.2.3` (consensus client). JSON-RPC provider yang sama = reth v2.7.0 (execution client, lihat [EVM 01.13 - Hasil Praktik](EVM%2001.13%20-%20Hasil%20Praktik.md)) |
| Beacon API `/eth/v2/beacon/blocks/head` | fork `fulu`, slot **15.399.156**, `proposer_index` **2.092.386**; body berisi 13 field, salah satunya `execution_payload` |
| Isi `execution_payload` | `block_number` 26.160.253, 200 tx, `gas_used` 17.877.391, `fee_recipient` `0x3963…aa49`, `block_hash` `0xa9a4…0556` |
| `cast block 26160253` lewat JSON-RPC | hash, `miner`, jumlah tx, dan `gasUsed` **sama persis** dengan `execution_payload` di atas |
| `parentBeaconBlockRoot` di blok execution | `0xd2ea…8b86` = `parent_root` di header beacon block → blok execution menyimpan penunjuk ke lapisan consensus (EIP-4788) |
| `timestamp` blok | 1.791.613.895 = genesis beacon 1.606.824.023 + 15.399.156 × 12 → satu slot tepat 12 detik |
| Validator 2.092.386 di Beacon API | status `active_ongoing`, effective balance **32 ETH**. Identitas ini tidak ada di blok execution |

## Yang tidak dicek

- Nama method Engine API dan alur di diagram sequence berasal dari pengetahuan umum. Port
  Engine API tidak terbuka untuk publik, jadi tidak bisa dicoba dari RPC publik.
