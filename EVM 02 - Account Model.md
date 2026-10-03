---
title: EVM 02 - Account Model
tags: [learning, evm, ethereum, bsc, on-chain]
updated: 2026-10-03
---

# Account Model EVM

Catatan dari Sesi 2 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Semua angka di sini hasil observasi langsung
di `anvil` lokal (Foundry 1.7.1, chain ID 31337) dan BSC mainnet, bukan kutipan dokumentasi.
Pembanding: Account Model Solana.

Project praktik: `~/Documents/riset/evm/account-model` (template `forge init`, kontrak `Counter`).

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

## EOA

Akun #0 `anvil` (`0xf39F…2266`): balance 10.000 ETH, nonce 0, code `0x`, storage kosong.

**Alamat tidak perlu dibuat.** Alamat acak yang belum pernah ada langsung bisa menerima 1 ETH.
Setiap alamat 20 byte dianggap sudah ada dengan semua field bernilai nol. Beda dengan Solana,
di mana akun harus dibuat dan dibayar rent-nya dulu.

**Nonce menghitung transaksi keluar, bukan masuk.** Setelah transfer: nonce pengirim 0 → 1,
nonce penerima tetap 0. Fungsinya mencegah replay.

**Transfer ETH polos = tepat 21.000 gas.** Biaya terukur 0,000021000000021 ETH = 21.000 ×
1,000000001 gwei.

**Mengirim calldata ke EOA tidak gagal.** Transaksinya sukses tapi tidak melakukan apa-apa —
tetap memakan nonce dan gas. Tidak ada kode yang bisa menolak.

## Contract account

### Alamat kontrak bisa diprediksi sebelum deploy

```
alamat = keccak256(rlp(alamat_deployer, nonce_deployer))[12:]
```

`cast compute-address 0xf39F…2266 --nonce 1` → `0xe7f1…0512`. `forge create` men-deploy ke
alamat yang **persis sama**. Alamat kontrak murni fungsi dari siapa yang deploy dan nonce-nya.

### Isi kontrak Counter setelah deploy

| | |
| --- | --- |
| code size | **481 byte** |
| nonce | **1** (bukan 0) |
| balance | 0 |
| storage slot 0 | `number` (awal 0) |

### Storage bisa dibaca mentah tanpa ABI

`setNumber(42)` → `cast storage <kontrak> 0` = `0x…002a` = 42. Setelah `increment()` → 43, sama
dengan `number()` lewat ABI. Ini cara membaca state kontrak walaupun tidak ada fungsi getter.

### Selector tertanam di bytecode

| Fungsi | Selector | Ada di bytecode? |
| --- | --- | --- |
| `number()` | `0x8381f58a` | ya |
| `setNumber(uint256)` | `0x3fb5c1cb` | ya |
| `increment()` | `0xd09de08a` | ya |

Awal bytecode adalah **dispatcher**: baca 4 byte pertama calldata, lompat ke fungsi yang
cocok. Itu sebabnya `0x70a08231` (`balanceOf`) di riset tokenomics bisa dipanggil ke token mana
pun — setiap ERC-20 punya selector yang sama di dispatcher-nya.

## Observasi di BSC mainnet

Akun-akun dari CAKE Diperiksa Ulang 2026-09-24 dan BNB Diperiksa Ulang 2026-09-24:

| Akun | code | nonce | Artinya |
| --- | ---: | ---: | --- |
| CAKE token `0x0E09…cE82` | 7.285 byte | 1 | kontrak |
| MasterChefV2 `0xa5f8…7652` | 15.151 byte | 1 | kontrak |
| Safe burn `0xceba…4a0e` | **171 byte** | 1 | **proxy** — slot 0 = `0x3e5c63644e683549055b9be8653de26e0b4cd36e` (Gnosis Safe 1.3.0) |
| `0x…dEaD` | 0 | **0** | EOA yang tidak pernah mengirim transaksi; memegang 16,5 jt BNB |
| Bot `0xd099…48d4` | 0 | **6.725** | EOA yang memicu mint-burn **mingguan** CAKE (koreksi Sesi 4: bukan harian) |

Tiga pelajaran:

1. **Burn ke `0xdead` = kirim ke EOA yang private key-nya tidak diketahui siapa pun.** Nonce 0
   membuktikan alamat itu tidak pernah bisa bertransaksi.
2. **Kontrak tidak bisa memulai transaksi.** Setiap transaksi diawali EOA. MasterChef cuma
   bergerak karena bot EOA `0xd099…` memanggilnya — 6.725 kali. (Sesi 4: bot ini memanggil Safe
   → MultiSend → Timelock → MasterChefV2 seminggu sekali, lihat [EVM 04 - ABI](EVM%2004%20-%20ABI.md).)
3. **Proxy kecil, logikanya di tempat lain.** Safe 171 byte menyimpan storage sendiri, tapi
   meminjam logika dari kontrak di slot 0. Detail di Sesi 17.

## Pengecualian modern: EIP-7702

"EOA = code kosong" tidak lagi mutlak sejak upgrade Pectra. EOA bisa **mendelegasikan** code-nya
ke kontrak lain.

Percobaan di `anvil`: akun #1 (`0x7099…79C8`) mendelegasikan ke `Counter`, sekaligus memanggil
`setNumber(7)` dalam satu transaksi.

| | Sebelum | Sesudah |
| --- | --- | --- |
| Tipe transaksi | — | `0x4` |
| code akun #1 | `0x` | `0xef0100e7f1725e7734ce288f8367e1bb143e90bb3f0512` (**23 byte**) |
| `number()` di akun #1 | — | **7** |
| `number()` di Counter asli | 43 | **43** (tidak berubah) |

- Code 23 byte itu bukan bytecode, tapi **penunjuk**: `0xef0100` + alamat kontrak tujuan.
- **Storage tetap milik EOA.** Akun #1 meminjam logika Counter, tapi state-nya tersimpan di
  akun #1 sendiri. Konsep yang sama dengan proxy.
- Transaksi delegasi memakan **dua nonce**: satu untuk transaksinya, satu untuk otorisasinya.

**Jebakan yang ditemui:** delegasi yang dikirim tanpa calldata **revert** saat estimasi gas.
Begitu delegasi berlaku, EOA langsung "menjadi" Counter, dan Counter tidak punya `receive()`.
Solusinya: gabungkan delegasi dengan panggilan fungsi (`cast send <eoa> "setNumber(uint256)" 7 --auth <kontrak>`).

## EVM vs Solana

| | EVM | Solana (Account Model Solana) |
| --- | --- | --- |
| Akun baru | otomatis ada, semua field nol | harus dibuat + bayar rent |
| Code dan data | di akun yang sama (`code` + `storage`) | dipisah: program account vs data account |
| Token | saldo disimpan di storage kontrak token (`balanceOf`) | token account terpisah per pemilik per mint |
| Pembeda tipe akun | isi field `code` | field `executable` + `owner` |
| Siapa memulai transaksi | selalu EOA | signer (keypair) |

## Perintah

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
