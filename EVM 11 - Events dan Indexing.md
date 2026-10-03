---
title: EVM 11 - Events dan Indexing
tags: [learning, evm, events, indexing, rpc]
updated: 2026-09-24
---

# Events dan Indexing

Catatan dari Sesi 11 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Indexer mini ditulis sendiri, diuji di `anvil`,
lalu dicoba ke RPC Ethereum publik. Lanjutan dari [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md).

Project: `~/Documents/riset/evm/indexer` — `indexer.py` (rebuild saldo dari log, fetch adaptif),
`make_activity.py` (60 transfer acak). Kontrak biaya: `solidity-basics/src/EventCost.sol`.

## Storage = keadaan sekarang, log = sejarah

`balanceOf` cuma bisa menjawab untuk alamat yang ditanyakan. **Log adalah satu-satunya cara
tahu siapa saja yang pernah memegang token** — itu cara explorer membuat daftar holder.

## Indexer mini

Aturan: tiap `Transfer(from, to, value)` → kurangi `from`, tambah `to`; `from = 0x0` = mint.

Hasil pada 70 log (1 mint + 69 transfer): **6 holder, semua cocok dengan `balanceOf`**, jumlah
saldo = jumlah mint.

### Temuan tak sengaja: alamat nyasar

Alamat anvil #3 yang ditulis dari ingatan salah (`0x90F7…dE33`, yang benar `0x90F7…b906`). Indexer
menemukannya: **113.370 token (11,3% supply) di alamat tanpa pemilik** — terbakar tanpa sengaja.
EVM menerima alamat 20 byte apa pun tanpa bertanya. Private key akun #4 dari ingatan juga salah.

**Perbaikan:** jangan tulis key/alamat dari ingatan. Turunkan dari mnemonic:

```bash
cast wallet private-key --mnemonic "test test test test test test test test test test test junk" --mnemonic-index 3
```

## Filter topic

`null` = apa saja, array = salah satu dari. Di data lokal (70 log):

| Filter | Arti | Log |
| --- | --- | ---: |
| `[T]` | semua Transfer | 70 |
| `[T, #0]` | dari #0 | 20 |
| `[T, null, #0]` | ke #0 | 13 |
| `[T, #0, #1]` | dari #0 ke #1 | 7 |
| `[T, null, [#0, #1]]` | ke #0 **atau** #1 | 30 |
| `[T, 0x0]` | **mint** | 1 |
| `[T, null, alamat nyasar]` | ke alamat nyasar | 3 |

Pola riset: `[T, null, 0xdead]` untuk burn CAKE/UNI, `[T, 0x0]` untuk mint CAKE
([EVM 04 - ABI](EVM%2004%20-%20ABI.md)). Tanpa `address` → Transfer dari **semua** kontrak.

## Batas `eth_getLogs` di RPC publik (Ethereum, UNI Transfer, 2026-09-24)

| RPC | Batas | Cara menolak |
| --- | --- | --- |
| `ethereum-rpc.publicnode.com` | 3.001 blok / 7.701 log lolos; 10.000 blok gagal — kemungkinan batas **jumlah log** | HTTP **403** |
| `eth.drpc.org` | < 1.000 blok | HTTP **400** tanpa pesan |
| `1rpc.io/eth` | **50 blok** | pesan jelas |
| `rpc.flashbots.net` | head tertinggal walau sudah −5 blok | `beyond current head` |
| nodereal BSC (riset CAKE) | 40.000 blok | — |

Jebakan yang ditemui:

- **HTTP 403 untuk User-Agent `Python-urllib`** (Cloudflare) → kirim User-Agent sendiri.
- **Load balancer**: node yang menjawab `eth_getLogs` bisa tertinggal dari node yang menjawab
  `eth_blockNumber` → jangan baca sampai ujung.
- Penolakan rentang **tidak seragam** (403, 400, JSON error) → pemecah rentang memperlakukan error
  apa pun sebagai "terlalu besar", dan baru menyerah di rentang 1 blok.

### Pemecah rentang adaptif

Gagal → rentang dibelah dua. Berhasil → rentang digandakan (maks 20.000).

publicnode: **5.000 blok → 12.489 log, 2 request, 22,5 detik** (2.000 lalu 3.001 blok).

UNI ≈ 2,4 Transfer/blok → setahun ≈ 6 jt log ≈ **~3 jam** dari RPC publik. Itu sebabnya riset
memakai Blockscout, dan proyek nyata memakai layanan indexing atau node sendiri.

## Reorg

`anvil_reorg(3, …)` mengganti 3 blok terakhir:

| | Sebelum | Sesudah |
| --- | --- | --- |
| Hash blok 68–70 | `0x061a…`, `0xdaa0…`, `0x2acc…` | **semua berubah** |
| Log Transfer di blok itu | 3 | **0** |
| Holder yang saldonya salah di index lama | — | **3** (contoh #1: index 121.451, sebenarnya 222.260) |

(Transaksi pengganti yang disisipkan tidak ikut masuk — kemungkinan nonce. Efek utamanya sama.)

Indexer yang benar:

1. Simpan **hash** tiap blok yang diproses, bukan cuma nomornya.
2. Tiap blok baru: cek `parentHash` cocok dengan hash tersimpan. Tidak cocok → mundur ke titik
   percabangan, buang data blok yang batal, index ulang.
3. Atau tunggu finality sebelum menganggap data pasti.

Riset tokenomics aman karena membaca data historis, jauh dari ujung chain.

## Event 40–50x lebih murah dari storage

Mencatat (from, to, value), transaksi nyata di anvil:

| Cara | Biaya pencatatan |
| --- | ---: |
| Storage, 3 slot (pertama / berikutnya) | 88.561 / 71.461 |
| **Event** `Recorded` (3 topic + 32 byte data) | **1.760** |

Rumus LOG: `375 + 375 × jumlah topic + 8 × byte data` → 375 + 1.125 + 256 = **1.756** ✓.

Harga murahnya ada syarat: **kontrak tidak bisa membaca event.** Kontrak menyimpan keadaan sekarang
(saldo), sejarahnya diserahkan ke log untuk dibaca indexer.

## Perintah

```bash
python3 indexer.py http://127.0.0.1:8545 <token>              # rebuild + cek saldo
cast logs --from-block 0 --address <token> 'Transfer(address indexed,address indexed,uint256)'
cast rpc anvil_reorg 3 '[["<raw-tx>", 0]]'                      # simulasi reorg
```
