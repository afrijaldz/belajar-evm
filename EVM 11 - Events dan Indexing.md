---
title: EVM 11 - Events dan Indexing
tags: [learning, evm, events, indexing, rpc]
updated: 2026-10-24
---

# Events dan Indexing

Sesi 11 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar kenapa log (event) adalah sumber sejarah
on-chain, cara membangun indexer sederhana dari log `Transfer`, cara memfilter log dengan topic,
batas `eth_getLogs` di RPC publik, efek reorg, dan kenapa event jauh lebih murah dari storage.
Lanjutan dari [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md).

Hasil percobaan untuk materi ini ada di [EVM 11 - Hasil Praktik](Hasil%20Praktik/EVM%2011%20-%20Hasil%20Praktik.md).

## Storage = keadaan sekarang, log = sejarah

`balanceOf` cuma bisa menjawab untuk alamat yang ditanyakan. **Log adalah satu-satunya cara
tahu siapa saja yang pernah memegang token** — itu cara explorer membuat daftar holder.

## Indexer mini

Aturan: tiap `Transfer(from, to, value)` → kurangi `from`, tambah `to`; `from = 0x0` = mint.

Setelah semua log diproses, saldo tiap holder harus sama dengan `balanceOf`, dan jumlah semua saldo
harus sama dengan jumlah mint. Kalau tidak sama, ada log yang terlewat.

**Jebakan:** EVM menerima alamat 20 byte apa pun tanpa bertanya. Satu karakter salah saat menulis
alamat dari ingatan, dan token terkirim ke alamat tanpa pemilik — terbakar tanpa sengaja. Indexer
akan memperlihatkannya sebagai holder asing. Jangan tulis key/alamat dari ingatan; turunkan dari
mnemonic:

```bash
cast wallet private-key --mnemonic "test test test test test test test test test test test junk" --mnemonic-index 3
```

## Filter topic

`null` = apa saja, array = salah satu dari. `T` = topic0 event `Transfer`.

| Filter | Arti |
| --- | --- |
| `[T]` | semua Transfer |
| `[T, A]` | dari A |
| `[T, null, A]` | ke A |
| `[T, A, B]` | dari A ke B |
| `[T, null, [A, B]]` | ke A **atau** B |
| `[T, 0x0]` | **mint** |

Pola yang sering dipakai: `[T, null, 0xdead]` untuk burn gaya B, `[T, 0x0]` untuk mint
([EVM 04 - ABI](EVM%2004%20-%20ABI.md)). Tanpa `address` → Transfer dari **semua** kontrak.

## Batas `eth_getLogs` di RPC publik

Setiap RPC publik membatasi `eth_getLogs`: ada yang membatasi rentang blok (dari puluhan sampai
puluhan ribu blok), ada yang membatasi jumlah log. Cara menolaknya juga berbeda: HTTP 403, HTTP 400
tanpa pesan, atau JSON error dengan pesan jelas.

**Jebakan:**

- **HTTP 403 untuk User-Agent `Python-urllib`** (Cloudflare) → kirim User-Agent sendiri.
- **Load balancer**: node yang menjawab `eth_getLogs` bisa tertinggal dari node yang menjawab
  `eth_blockNumber` (`beyond current head`) → jangan baca sampai ujung.
- Penolakan rentang **tidak seragam** (403, 400, JSON error) → pemecah rentang memperlakukan error
  apa pun sebagai "terlalu besar", dan baru menyerah di rentang 1 blok.

### Pemecah rentang adaptif

Gagal → rentang dibelah dua. Berhasil → rentang digandakan (maks 20.000).

Untuk token yang ramai, membaca log setahun dari RPC publik bisa makan waktu berjam-jam. Itu
sebabnya analisis memakai API explorer seperti Blockscout, dan proyek nyata memakai layanan indexing
atau node sendiri.

## Reorg

Saat reorg, beberapa blok terakhir diganti. Hash blok-blok itu berubah, dan log di dalamnya bisa
hilang atau berbeda. Indexer yang sudah memproses blok lama akan menyimpan saldo yang salah.

Indexer yang benar:

1. Simpan **hash** tiap blok yang diproses, bukan cuma nomornya.
2. Tiap blok baru: cek `parentHash` cocok dengan hash tersimpan. Tidak cocok → mundur ke titik
   percabangan, buang data blok yang batal, index ulang.
3. Atau tunggu finality sebelum menganggap data pasti.

Membaca data historis yang jauh dari ujung chain aman dari reorg.

## Event jauh lebih murah dari storage

Rumus LOG: `375 + 375 × jumlah topic + 8 × byte data`. Event dengan 3 topic + 32 byte data:
375 + 1.125 + 256 = **1.756** gas. Menyimpan data yang sama di 3 slot storage butuh puluhan ribu gas
(SSTORE), jadi event sekitar 40–50x lebih murah.

Harga murahnya ada syarat: **kontrak tidak bisa membaca event.** Kontrak menyimpan keadaan sekarang
(saldo), sejarahnya diserahkan ke log untuk dibaca indexer.

## Coba sendiri

1. Deploy token di `anvil`, lalu buat puluhan transfer acak antar akun.
2. Tulis indexer yang membangun ulang saldo dari log `Transfer`, lalu bandingkan dengan `balanceOf`.
3. Coba filter topic di tabel atas dan hitung jumlah log tiap filter.
4. Coba `eth_getLogs` ke beberapa RPC publik dengan rentang besar, dan catat cara masing-masing
   menolak.
5. Simulasikan reorg dengan `anvil_reorg`, lalu bandingkan hash blok, log, dan saldo indexer.
6. Bandingkan gas mencatat (from, to, value) di storage vs lewat event.

```bash
python3 indexer.py http://127.0.0.1:8545 <token>              # rebuild + cek saldo
cast logs --from-block 0 --address <token> 'Transfer(address indexed,address indexed,uint256)'
cast rpc anvil_reorg 3 '[["<raw-tx>", 0]]'                      # simulasi reorg
```
