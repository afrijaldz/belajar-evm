---
title: EVM 02 - Account Model
tags: [learning, evm, ethereum, bsc, on-chain]
updated: 2026-10-24
---

# Account Model EVM

Sesi 2 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar apa saja isi sebuah akun EVM, apa
beda EOA dan contract account, dan bagaimana alamat kontrak ditentukan.

Hasil percobaan untuk materi ini ada di [EVM 02 - Hasil Praktik](Hasil%20Praktik/EVM%2002%20-%20Hasil%20Praktik.md).

## Satu kalimat yang menjelaskan semuanya

**Setiap alamat EVM punya empat field yang sama — `balance`, `nonce`, `code`, `storage` — dan
"EOA" atau "kontrak" cuma soal apakah `code` kosong atau tidak.** Tidak ada field "tipe akun".

| Field | EOA | Contract |
| --- | --- | --- |
| `balance` | ada | ada (kontrak juga bisa memegang ETH/BNB) |
| `nonce` | jumlah transaksi yang **dikirim** | mulai dari **1** (EIP-161), menghitung kontrak yang ia buat |
| `code` | kosong (`0x`) | bytecode, tidak bisa diubah setelah deploy |
| `storage` | selalu kosong | slot 32-byte, key → value |

### Penjelasan detail

1. [EVM 02.01 - EOA dan Contract Account](EVM%2002%20-%20Detail/EVM%2002.01%20-%20EOA%20dan%20Contract%20Account.md) — Apa itu EOA dan contract account, siapa yang mengendalikan, cara lahir, cara membedakan
2. [EVM 02.02 - Nonce](EVM%2002%20-%20Detail/EVM%2002.02%20-%20Nonce.md) — Nomor urut akun: anti-replay, urutan tx, ganti tx di mempool, nonce kontrak, alamat kontrak

## EOA

**Alamat tidak perlu dibuat.** Setiap alamat 20 byte dianggap sudah ada dengan semua field
bernilai nol, jadi alamat yang belum pernah dipakai pun langsung bisa menerima ETH.

**Nonce menghitung transaksi keluar, bukan masuk.** Setiap kali sebuah akun mengirim transaksi,
nonce-nya naik 1. Menerima transaksi tidak mengubah nonce. Fungsinya mencegah replay: transaksi
yang sama tidak bisa dijalankan dua kali.

**Transfer ETH polos = tepat 21.000 gas.** Biayanya 21.000 × harga gas.

**Mengirim calldata ke EOA tidak gagal.** EOA tidak punya kode, jadi tidak ada yang bisa menolak.
Transaksinya sukses tapi tidak melakukan apa-apa — tetap memakan nonce dan gas.

## Contract account

### Alamat kontrak bisa diprediksi sebelum deploy

```
alamat = keccak256(rlp(alamat_deployer, nonce_deployer))[12:]
```

Alamat kontrak murni fungsi dari siapa yang deploy dan nonce-nya saat itu. `cast compute-address`
menghitungnya sebelum deploy, dan `forge create` akan men-deploy ke alamat yang persis sama.

### Nonce kontrak mulai dari 1

Sejak EIP-161, kontrak yang baru di-deploy langsung punya nonce 1, bukan 0. Nonce ini naik setiap
kali kontrak itu membuat kontrak lain.

### Storage bisa dibaca mentah tanpa ABI

Storage kontrak berupa slot 32 byte. Variabel pertama di kontrak biasanya ada di slot 0.
`cast storage <kontrak> <slot>` membaca isi slot langsung, jadi state kontrak bisa dibaca
walaupun kontrak itu tidak punya fungsi getter.

### Selector tertanam di bytecode

Setiap fungsi publik punya **selector** 4 byte (lihat [EVM 04 - ABI](EVM%2004%20-%20ABI.md)). Awal bytecode kontrak
adalah **dispatcher**: ia membaca 4 byte pertama calldata, lalu lompat ke fungsi yang cocok.

Karena selector hanya bergantung pada nama dan tipe parameter fungsi, `balanceOf(address)` selalu
`0x70a08231`. Itu sebabnya `balanceOf` bisa dipanggil ke token ERC-20 mana pun.

### Kontrak tidak bisa memulai transaksi

Setiap transaksi selalu diawali EOA. Kontrak hanya bergerak kalau dipanggil, entah langsung oleh
EOA atau oleh kontrak lain yang dipanggil EOA. Kontrak yang "jalan otomatis" sebenarnya dipicu
oleh bot EOA.

## Dua pola yang sering ditemui

**Burn ke `0xdead`.** Alamat `0x000000000000000000000000000000000000dEaD` adalah EOA yang private
key-nya tidak diketahui siapa pun. Nonce-nya 0, artinya alamat itu tidak pernah bisa mengirim
transaksi. Token yang dikirim ke sana praktis hilang. Detail di [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md).

**Proxy.** Kontrak proxy berukuran kecil: ia menyimpan storage sendiri, tapi meminjam logika dari
kontrak lain yang alamatnya disimpan di storage-nya. Detail di [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md).

## Pengecualian modern: EIP-7702

"EOA = code kosong" tidak lagi mutlak sejak upgrade Pectra. EOA bisa **mendelegasikan** code-nya
ke kontrak lain lewat transaksi tipe `0x4`.

- Setelah delegasi, code EOA berisi 23 byte. Itu bukan bytecode, tapi **penunjuk**: `0xef0100` +
  alamat kontrak tujuan.
- **Storage tetap milik EOA.** EOA meminjam logika kontrak tujuan, tapi state-nya tersimpan di EOA
  sendiri. Konsepnya sama dengan proxy.
- Transaksi delegasi memakan **dua nonce**: satu untuk transaksinya, satu untuk otorisasinya.

**Jebakan:** delegasi yang dikirim tanpa calldata bisa **revert** saat estimasi gas. Begitu
delegasi berlaku, EOA langsung "menjadi" kontrak tujuan. Kalau kontrak itu tidak punya
`receive()`, transaksi tanpa calldata gagal. Solusinya: gabungkan delegasi dengan panggilan
fungsi (`cast send <eoa> "setNumber(uint256)" 7 --auth <kontrak>`).

## Coba sendiri

Jalankan `anvil`, lalu:

1. Cek akun #0: `cast balance`, `cast nonce`, `cast code`. Code harus `0x`.
2. Kirim ETH ke alamat acak, lalu cek nonce pengirim dan penerima.
3. Hitung alamat kontrak dengan `cast compute-address`, deploy `Counter` dengan `forge create`,
   lalu bandingkan alamatnya.
4. Panggil `setNumber(42)`, lalu baca slot 0 dengan `cast storage`.
5. Cari selector `number()` (`cast sig 'number()'`) di output `cast code`.

```bash
anvil --port 8545                                  # chain lokal, 10 akun @ 10.000 ETH
export ETH_RPC_URL=http://127.0.0.1:8545
cast balance <addr> --ether ; cast nonce <addr> ; cast code <addr> ; cast codesize <addr>
cast storage <addr> <slot>                         # slot mentah, tanpa ABI
cast compute-address <deployer> --nonce <n>        # prediksi alamat kontrak
forge create src/Counter.sol:Counter --private-key <pk> --broadcast
cast sig 'number()'                                # selector 4 byte
cast send <eoa> "fn(...)" <args> --auth <kontrak> --private-key <pk-eoa>   # EIP-7702
```
