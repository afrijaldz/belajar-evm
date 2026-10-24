---
title: EVM 03 - Transaksi dan Gas
tags: [learning, evm, ethereum, bsc, gas]
updated: 2026-10-24
---

# Transaksi dan Gas EVM

Sesi 3 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), lanjutan dari [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md). Di sesi ini kamu
belajar isi sebuah transaksi, ke mana biaya gas pergi, bagaimana base fee naik-turun, dan
operasi apa yang paling mahal.

Hasil percobaan untuk materi ini ada di [EVM 03 - Hasil Praktik](Hasil%20Praktik/EVM%2003%20-%20Hasil%20Praktik.md).

## Transaksi = data kecil yang ditandatangani

Transaksi hanyalah sekumpulan field yang di-encode lalu ditandatangani. `cast mktx` membuat dan
menandatangani transaksi **tanpa mengirimnya**, jadi kamu bisa melihat bentuk mentahnya. Transfer
ETH polos tipe EIP-1559 hanya sekitar 119 byte. Byte pertama = tipe transaksi (`0x02` untuk
EIP-1559).

| Field | Arti |
| --- | --- |
| `chainId` | mencegah replay di chain lain |
| `nonce` | urutan transaksi pengirim |
| `gas` | **batas** gas |
| `maxFeePerGas` | harga maksimal per gas |
| `maxPriorityFeePerGas` | tip maksimal untuk validator |
| `to`, `value`, `input` | tujuan, jumlah, calldata |
| `r`, `s`, `yParity` | tanda tangan |

**Tidak ada field `from`.** Pengirim dihitung ulang dari tanda tangan — tidak bisa dipalsukan.

## EIP-1559: ke mana biaya pergi

Biaya per gas terdiri dari dua bagian: **base fee** (ditetapkan protokol) dan **priority fee**
(tip). Contoh transfer ETH polos (21.000 gas) dengan base fee 1 gwei dan tip 2 gwei:

| Bagian | Rumus | Jumlah | Ke mana |
| --- | --- | ---: | --- |
| Base fee | 1 gwei × 21.000 | 21.000 gwei | **dibakar** |
| Priority fee | 2 gwei × 21.000 | 42.000 gwei | validator (alamat `coinbase` blok) |
| **Total** | 3 gwei × 21.000 | **63.000 gwei** | |

- **`maxFeePerGas` cuma plafon.** Walaupun maxFee dipasang jauh lebih tinggi, yang dibayar tetap
  base fee + tip.
- **maxFee di bawah base fee langsung ditolak** (`max fee per gas less than block base fee`),
  tidak masuk blok.

Ini burn di level protokol, bukan dari kontrak: bagian base fee dari setiap transaksi dimusnahkan.

## Base fee diatur rumus, bukan lelang

```
base_baru = base_lama × (1 + 1/8 × (gas_terpakai − target) / target)      target = gasLimit / 2
```

- Blok tepat di target → base fee tetap.
- Blok kosong → base fee turun 12,5% (×0,875). Ini penurunan maksimal per blok.
- Blok hampir penuh → base fee naik mendekati 12,5% (blok penuh = naik maksimal 12,5%).
- Hasil hitungan bisa selisih 1 wei karena pembulatan bilangan bulat.

Transaksi yang butuh gas melebihi block gas limit ditolak saat estimasi.

## Harga calldata: 10 dan 40 gas per byte

Untuk transaksi yang dominan data (misalnya transaksi yang hanya membawa calldata tanpa
eksekusi), harganya **10 gas per byte nol** dan **40 gas per byte bukan nol** — bukan 4 dan 16
seperti di banyak tutorial lama. Penyebabnya **harga minimum EIP-7623** (Pectra): 10 gas per
"token", byte nol = 1 token, byte bukan nol = 4 token.

Artinya byte nol jauh lebih murah daripada byte bukan nol.

## Storage adalah operasi termahal

Biaya `SSTORE` bergantung pada nilai lama dan nilai baru di slot itu:

| Operasi pada satu slot | Biaya `SSTORE` |
| --- | --- |
| **0 → bukan nol** (slot baru) | **20.000** |
| bukan nol → bukan nol lain (ubah) | 2.900 |
| nilai sama ditulis ulang | lebih murah dari "ubah" |
| bukan nol → 0 (hapus) | dapat **refund 4.800** |

Membuat satu slot baru (20.000) ≈ satu transfer ETH penuh. Itu sebabnya transfer token ke
alamat yang belum pernah punya saldo lebih mahal.

## Gagal tetap bayar

| Kasus | status | Gas yang dibayar | Nonce | State |
| --- | --- | --- | --- | --- |
| Revert (mis. `require` gagal) | `false` | gas yang **terpakai** sampai titik revert | naik | tidak berubah |
| Out of gas | `false` | **seluruh gas limit** | naik | tidak berubah |

- Revert: yang dibayar hanya gas yang terpakai sampai titik revert.
- Out of gas: **seluruh gas limit hangus**.
- Alasan revert bisa dibaca ulang dengan `cast run <tx>` (contoh output: `[Revert] amount is zero`).

## Receipt dan log

Setiap `emit` di kontrak menghasilkan satu **log** di receipt transaksi. Contoh event
`Added(address indexed by, uint256 amount, uint256 newTotal)`:

| Bagian log | Isi | Arti |
| --- | --- | --- |
| `address` | alamat kontrak | kontrak yang memancarkan |
| `topics[0]` | `keccak256("Added(address,uint256,uint256)")` | identitas event |
| `topics[1]` | nilai `by`, dipad 32 byte | parameter `indexed` — bisa difilter |
| `data` | `amount`, `newTotal` | parameter non-indexed |

**Contoh pemakaian: mencari burn token.** Event ERC-20 `Transfer(address indexed from, address
indexed to, uint256 value)` punya `topics[0]` = `0xddf252ad…` =
`keccak256("Transfer(address,address,uint256)")`. Filter `eth_getLogs` dengan topics
`[T, null, DEAD]` berarti "Transfer, pengirim siapa saja, penerima `0xdead`". Ini bisa karena
`from` dan `to` bertipe `indexed`.

## Ethereum vs BSC

- **Base fee BSC selalu 0.** BSC memakai format transaksi EIP-1559, tapi tidak ada base fee yang
  dibakar. Burn BNB datang dari mekanisme lain: BEP-95 dan auto-burn kuartalan.
- Blok BSC jauh lebih sering (sekitar 0,45 detik) dibanding Ethereum (12 detik), jadi jumlah
  transaksi per blok tidak bisa dibandingkan langsung.
- Transaksi bertipe `0x0` (legacy) masih dipakai di BSC.

## Jebakan

- `cast block <n> gasLimit` salah — yang benar `cast block <n> --field gasLimit`.
- `jq tonumber` tidak bisa membaca hex (`0x5208`). Pakai Python `int(x, 16)` atau `cast to-dec`.
- Aritmetika zsh `$(( ))` overflow di atas 2^63 — saldo dalam wei (≈10^22) jadi salah. Pakai Python.
- Transaksi yang akan revert tidak bisa dikirim lewat `cast send` biasa karena estimasi gas gagal
  duluan. Pasang `--gas-limit` manual supaya tetap terkirim.

## Coba sendiri

Jalankan `anvil` dengan block gas limit kecil supaya base fee mudah bergerak, lalu:

1. Buat transaksi dengan `cast mktx`, decode dengan `cast decode-transaction`, kirim dengan
   `cast publish`.
2. Bandingkan `gasUsed` dan `effectiveGasPrice` di receipt dengan rumus EIP-1559 di atas.
3. Tambang blok kosong (`anvil_mine`) beberapa kali, lalu cek `baseFeePerGas` turun 12,5% per blok.
4. Kirim calldata 100 byte `0x00` dan 100 byte `0xff`, bandingkan `gasUsed`-nya.
5. Tulis kontrak kecil dengan satu variabel `uint256`, lalu ukur gas untuk 0 → nilai, ubah nilai,
   nilai sama, dan nilai → 0.
6. Buat fungsi dengan `require` dan event. Picu revert dan out of gas (dengan `--gas-limit`), lalu
   baca alasannya dengan `cast run`.

```bash
anvil --port 8545 --gas-limit 100000                     # block kecil supaya base fee mudah bergerak
cast mktx <to> --value 1ether --private-key <pk> --nonce 0 --gas-limit 21000 \
  --priority-gas-price 2gwei --gas-price 3gwei --chain 31337
cast decode-transaction <raw>
cast publish <raw>
cast receipt <tx> gasUsed ; cast receipt <tx> effectiveGasPrice
cast block latest --field baseFeePerGas
cast rpc anvil_mine 1                                    # blok kosong
cast rpc anvil_setBlockGasLimit 30000000
cast run <tx>                                            # trace + alasan revert
cast keccak 'Transfer(address,address,uint256)'          # topic[0] event
```
