---
title: EVM 04 - ABI
tags: [learning, evm, ethereum, bsc, abi]
updated: 2026-09-24
---

# ABI EVM

Catatan dari Sesi 4 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Semua hasil observasi langsung dengan `cast`
(Foundry 1.7.1), `anvil` lokal, dan satu transaksi nyata di BSC. Lanjutan dari
[EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md).

Project praktik: `~/Documents/riset/evm/account-model` (kontrak `AbiLab` dengan event indexed
string dan event anonymous).

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

4 byte = cuma ~4,3 miliar kemungkinan, jadi tabrakan bisa dicari dengan sengaja. `cast 4byte
0x42966c68` mengembalikan **dua nama**. Nama fungsi dari database selector publik (juga yang
ditampilkan explorer untuk kontrak tidak terverifikasi) hanyalah tebakan.

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

`Note(address indexed by, string indexed tag, string text, uint256[] values)`:

| Bagian | Isi |
| --- | --- |
| `topics[0]` | `keccak256("Note(address,string,string,uint256[])")` |
| `topics[1]` | `by` (address, dipad 32 byte) |
| `topics[2]` | **`keccak256("burn")`** — bukan teks "burn" |
| `data` | `text` dan `values` dengan skema head + tail (offset `0x40`, `0x80`) |

- **`indexed` tipe dinamis disimpan sebagai hash.** Bisa difilter dengan nilai persis, tapi
  teks aslinya tidak bisa dibaca balik dari log.
- **Event `anonymous` tidak punya `topics[0]`** — `Ping(99)` menghasilkan `topics: []`. Tidak bisa
  difilter berdasarkan nama event.
- Maksimal 4 topic (1 signature + 3 indexed).
- **Status indexed tidak ikut di-hash ke `topics[0]`.** Signature yang sama bisa punya susunan
  indexed berbeda di kontrak lain. Decode log yang benar butuh **ABI kontraknya**, bukan cuma
  database selector.

## Studi kasus: transaksi mint-burn CAKE

Transaksi `0xb7a7e2989258cde2b25942484f051d3168e65b9db0d0d11f750f211d780abd11` (BSC,
2026-09-21 06:51 UTC) dari CAKE Diperiksa Ulang 2026-09-24. Calldata 1.476 byte.

### Lapisan calldata

```
EOA bot 0xd099… (nonce 6.725)
 └─ Safe 0xeCc9….execTransaction (selector 0x6a761202)
      tanda tangan 195 byte = 3 × 65 byte → 3 signer
      operation = 1 (delegatecall)
     └─ MultiSend 0x40A2….multiSend(bytes) (0x8d80ff0a)
          format packed: op (1) | to (20) | value (32) | panjang (32) | data
         ├─ Timelock 0xa1f4….executeTransaction(address,uint256,string,bytes,uint256) (0x0825f38f)
         │    target MasterChefV2, signature "burnCake(bool)", data true, eta 2026-09-21 01:00 UTC
         └─ Timelock 0xa1f4….queueTransaction(...) (0x3a66f901)
              target MasterChefV2, signature "burnCake(bool)", data true, eta 2026-09-28 01:00 UTC
```

- **Siklusnya mingguan.** Selisih eta = 604.800 detik = 7 hari. Tiap minggu multisig
  menjalankan perintah yang diantrekan minggu lalu dan mengantrekan perintah minggu depan.
- **Timelock memaksa jeda 7 hari**, jadi perintah minggu depan sudah bisa dibaca on-chain hari
  ini lewat `queueTransaction`.
- Timelock gaya Compound menerima **`string signature`** (`"burnCake(bool)"`) dan menghitung
  selector sendiri.
- **Catatan MultiSend:** di dalam `bytes` ada format *packed* (bukan ABI standar) — tiap
  sub-transaksi disambung tanpa padding. Harus di-parse manual.

### 17 log

| Log | Emitter | Event | Arti |
| ---: | --- | --- | --- |
| 0–1 | Safe + `0x53a1…78f1` | `SafeMultiSigTransaction`, `SafeTxData` | catatan eksekusi multisig |
| 2–8 | MasterChefV2 | 7× `UpdatePool` | 7 pool dummy diperbarui |
| 9 | CAKE | `Transfer` 0x0 → Safe burn `0xceba…`, **530.328** | mint 10% dev share |
| 10 | CAKE | `Transfer` 0x0 → SyrupBar `0x009c…`, **5.303.280** | mint 90% |
| 11 | CAKE | `Transfer` SyrupBar → MasterChefV2, 5.303.280 | diteruskan |
| 12 | MasterChefV1 `0x73fe…` | `Deposit(user=MCv2, pid=526, amount=0)` | amount 0 = **panen**, bukan setor |
| 13 | CAKE | `Transfer` MasterChefV2 → Safe burn, **54.049.639** | `burnCake` |
| 14–15 | Timelock | `ExecuteTransaction`, `QueueTransaction` | eksekusi + antre |
| 16 | Safe | `ExecutionSuccess` | selesai |

Terverifikasi: `0x009c…` = `"SyrupBar Token"`; `MasterChefV1.cakePerBlock()` = **40 CAKE**.

**Belum cocok:** di transaksi ini MCv2 cuma menerima 5,3 jt tapi mengirim 54,0 jt ke Safe burn,
padahal saldo MCv2 di titik-titik sampel selalu kecil (~0,37 jt). Kemungkinan sisanya dipanen
oleh transaksi lain sepanjang minggu — belum dibuktikan. Jawab di Sesi 16.

## Koreksi ke catatan sebelumnya

[EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md) menyebut bot `0xd099…` memicu mint-burn **harian**. Yang benar
**mingguan** — sudah diperbaiki di sana dan di CAKE Diperiksa Ulang 2026-09-24.

## Perintah

```bash
cast sig 'transfer(address,uint256)'
cast keccak 'Transfer(address,address,uint256)'
cast calldata 'f(uint256,string,uint256[])' 7 "halo CAKE" "[10,20,30]"
cast calldata-decode '<sig>' <calldata>
cast 4byte <selector> ; cast 4byte-event <topic0> ; cast 4byte-calldata <calldata>
cast decode-abi 'x()(string,uint256[])' <log data>
cast tx <hash> input --rpc-url <rpc> ; cast receipt <hash> --json --rpc-url <rpc>
```

Jebakan tooling: argumen negatif ke `cast calldata` perlu hati-hati (`-1` bisa dibaca sebagai
flag); `pkill -f <pola>` bisa ikut membunuh shell yang menjalankannya kalau polanya ada di
command line shell itu sendiri.
