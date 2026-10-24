---
title: EVM 04 - ABI
tags: [learning, evm, ethereum, bsc, abi]
updated: 2026-10-24
---

# ABI EVM

Sesi 4 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), lanjutan dari [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md). Di sesi ini kamu
belajar bagaimana nama fungsi, argumen, dan event diubah menjadi byte: selector, encoding
calldata, dan struktur log.

Hasil percobaan untuk materi ini ada di [EVM 04 - Hasil Praktik](Hasil%20Praktik/EVM%2004%20-%20Hasil%20Praktik.md).

## Selector

**Selector = 4 byte pertama `keccak256(signature)`.**

```
keccak256("transfer(address,uint256)") = 0xa9059cbb2ab0…  → selector 0xa9059cbb
```

| Signature | Selector | Pelajaran |
| --- | --- | --- |
| `transfer(address,uint256)` | `0xa9059cbb` | bentuk kanonik |
| `transfer(address,uint)` | `0x6cb927d8` | `uint` ≠ `uint256` di hash |
| `transfer(address, uint256)` | `0x9d61d234` | spasi mengubah hash |
| `burn(uint256)` | `0x42966c68` | **tabrakan** ↓ |
| `collate_propagate_storage(bytes16)` | `0x42966c68` | **tabrakan** ↑ |

4 byte = cuma ~4,3 miliar kemungkinan, jadi tabrakan bisa dicari dengan sengaja. Untuk selector
`0x42966c68`, `cast 4byte` mengembalikan **dua nama**. Nama fungsi dari database selector publik
(juga yang ditampilkan explorer untuk kontrak tidak terverifikasi) hanyalah tebakan.

## Encoding argumen statis

Setiap argumen statis = **tepat satu word 32 byte**, berapa pun ukuran tipenya.

| Tipe | Cara diisi | Contoh |
| --- | --- | --- |
| `address`, `uint`, `bool` | rata kanan, pad nol di kiri | `…dead`, `…01` |
| `int` negatif | two's complement, pad `ff` | `-1` = `ffff…ffff` |
| `bytesN` | **rata kiri**, pad nol di kanan | `a9059cbb0000…` |
| `uint8` | tetap 32 byte | 255 = `…00ff` |

`transfer(0xdead, 1e18)` selalu 4 + 32 + 32 = **68 byte**. `uint8` tidak lebih hemat di
calldata; padding nol murah karena byte nol lebih murah (lihat [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)).

## Encoding argumen dinamis: head + tail

`f(uint256 7, string "halo CAKE", uint256[] [10,20,30])` = **292 byte**:

| Byte | Isi | Arti |
| ---: | --- | --- |
| 0 | `…07` | HEAD arg0: nilai langsung |
| 32 | `…60` | HEAD arg1: **offset** ke tail string = 96 |
| 64 | `…a0` | HEAD arg2: **offset** ke tail array = 160 |
| 96 | `…09` | TAIL string: panjang 9 byte |
| 128 | `68616c6f2043414b45…00` | TAIL string: isi, rata kiri |
| 160 | `…03` | TAIL array: panjang 3 elemen |
| 192–256 | `…0a`, `…14`, `…1e` | TAIL array: 10, 20, 30 |

- Head = satu word per argumen. Statis ditulis langsung, dinamis diganti **offset**.
- Offset dihitung dari awal argumen (setelah selector).
- Tail = panjang, lalu isi yang dipad ke kelipatan 32.

## Decode

```bash
cast calldata-decode 'f(uint256,string,uint256[])' <calldata>   # signature diketahui
cast 4byte <selector>                                           # tebak nama dari database
cast 4byte-calldata <calldata>                                  # tebak + decode sekaligus
cast decode-abi 'x()(string,uint256[])' <data>                  # decode data log
```

## Event

Contoh event `Note(address indexed by, string indexed tag, string text, uint256[] values)`
yang dipancarkan dengan `tag = "burn"`:

| Bagian | Isi |
| --- | --- |
| `topics[0]` | `keccak256("Note(address,string,string,uint256[])")` |
| `topics[1]` | `by` (address, dipad 32 byte) |
| `topics[2]` | **`keccak256("burn")`** — bukan teks "burn" |
| `data` | `text` dan `values` dengan skema head + tail (offset `0x40`, `0x80`) |

- **`indexed` tipe dinamis disimpan sebagai hash.** Bisa difilter dengan nilai persis, tapi
  teks aslinya tidak bisa dibaca balik dari log.
- **Event `anonymous` tidak punya `topics[0]`** — misalnya event anonymous `Ping(99)` tanpa
  parameter indexed menghasilkan `topics: []`. Tidak bisa difilter berdasarkan nama event.
- Maksimal 4 topic (1 signature + 3 indexed).
- **Status indexed tidak ikut di-hash ke `topics[0]`.** Signature yang sama bisa punya susunan
  indexed berbeda di kontrak lain. Decode log yang benar butuh **ABI kontraknya**, bukan cuma
  database selector.

## Calldata berlapis: multisig, MultiSend, Timelock

Transaksi admin di protokol nyata jarang berupa satu panggilan langsung. Biasanya calldata-nya
berlapis, dan setiap lapisan perlu di-decode satu per satu:

```
EOA (bot / signer)
 └─ Safe.execTransaction (selector 0x6a761202)
      tanda tangan = N × 65 byte → N signer
      operation = 1 (delegatecall)
     └─ MultiSend.multiSend(bytes) (0x8d80ff0a)
          format packed: op (1) | to (20) | value (32) | panjang (32) | data
         ├─ Timelock.executeTransaction(address,uint256,string,bytes,uint256) (0x0825f38f)
         └─ Timelock.queueTransaction(...) (0x3a66f901)
```

- **Safe (Gnosis Safe)** adalah multisig. Panjang tanda tangan dibagi 65 byte = jumlah signer
  yang ikut menandatangani.
- **MultiSend** menggabungkan beberapa sub-transaksi dalam satu `bytes`. Isinya format *packed*
  (bukan ABI standar) — tiap sub-transaksi disambung tanpa padding. Harus di-parse manual.
- **Timelock** memaksa jeda antara "antre" (`queueTransaction`) dan "eksekusi"
  (`executeTransaction`). Karena perintah harus diantrekan dulu, perintah yang akan dieksekusi
  nanti sudah bisa dibaca on-chain sekarang.
- Timelock gaya Compound menerima **`string signature`** (misalnya `"burnCake(bool)"`) dan
  menghitung selector sendiri.
- Untuk membaca log transaksi seperti ini, cocokkan setiap log dengan kontrak yang
  memancarkannya (`address`), lalu decode dengan ABI kontrak itu. Event `Deposit` dengan
  `amount = 0` di kontrak farming gaya MasterChef biasanya berarti **panen**, bukan setor.

## Jebakan

- Argumen negatif ke `cast calldata` perlu hati-hati: `-1` bisa dibaca sebagai flag.
- `pkill -f <pola>` bisa ikut membunuh shell yang menjalankannya kalau polanya ada di command line
  shell itu sendiri.

## Coba sendiri

1. Hitung selector beberapa signature dengan `cast sig`, termasuk versi dengan `uint` dan spasi.
2. Cari nama untuk selector `0x42966c68` dengan `cast 4byte`.
3. Encode `f(uint256,string,uint256[])` dengan `cast calldata`, lalu pecah per 32 byte dan cocokkan
   dengan tabel head + tail di atas.
4. Tulis kontrak dengan event yang punya `string indexed` dan event `anonymous`, panggil di `anvil`,
   lalu lihat topics-nya di receipt.
5. Ambil satu transaksi multisig nyata dengan `cast tx <hash> input`, lalu decode lapis demi lapis.

```bash
cast sig 'transfer(address,uint256)'
cast keccak 'Transfer(address,address,uint256)'
cast calldata 'f(uint256,string,uint256[])' 7 "halo CAKE" "[10,20,30]"
cast calldata-decode '<sig>' <calldata>
cast 4byte <selector> ; cast 4byte-event <topic0> ; cast 4byte-calldata <calldata>
cast decode-abi 'x()(string,uint256[])' <log data>
cast tx <hash> input --rpc-url <rpc> ; cast receipt <hash> --json --rpc-url <rpc>
```
