---
title: EVM 02.02 - Nonce
tags: [learning, evm, account, nonce]
updated: 2026-10-03
---

# Nonce

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) · ← [EVM 02.01 - EOA dan Contract Account](EVM%2002.01%20-%20EOA%20dan%20Contract%20Account.md)

**Nonce adalah nomor urut di setiap akun.** Untuk EOA, nonce = jumlah transaksi yang sudah
dikirim. Untuk contract account, nonce = jumlah kontrak yang sudah ia buat, ditambah 1.

Kata "nonce" berasal dari *number used once*: setiap nilai hanya boleh dipakai sekali.

## Analogi: nomor cek di buku cek

Bayangkan buku cek dengan nomor urut 0, 1, 2, 3, …

- Bank hanya menerima cek dengan nomor **berikutnya**. Cek nomor 5 tidak bisa dicairkan
  sebelum cek nomor 4.
- Cek yang sudah dicairkan tidak bisa dicairkan lagi. Fotokopi cek nomor 3 ditolak karena
  nomor 3 sudah terpakai.
- Cek yang ditolak isinya (misalnya saldo kurang) tetap menghabiskan nomornya.

Di EVM, "bank" adalah node, "cek" adalah transaksi yang sudah ditandatangani, dan "nomor cek"
adalah field `nonce` di dalam transaksi.

```
Akun A (nonce sekarang = 3)

  tx nonce 2  ──► ditolak: "nonce too low"     (sudah terpakai → anti-replay)
  tx nonce 3  ──► dieksekusi, nonce A jadi 4
  tx nonce 5  ──► menunggu di mempool          (nonce 4 belum ada)
  tx nonce 4  ──► dieksekusi, lalu nonce 5 ikut dieksekusi
```

## Tiga fungsi nonce

1. **Mencegah replay.** Transaksi yang sudah ditandatangani bisa disalin siapa saja dari
   mempool atau block explorer. Tanpa nonce, orang lain bisa mengirim ulang transaksi
   "kirim 1 ETH ke B" berkali-kali sampai saldo habis. Karena nonce di dalam transaksi ikut
   ditandatangani, salinan transaksi itu langsung ditolak setelah transaksi asli masuk blok.
2. **Menentukan urutan.** Transaksi dari satu akun dieksekusi persis sesuai urutan nonce,
   walaupun dikirim tidak berurutan.
3. **Menentukan alamat kontrak.** Alamat kontrak = `keccak256(rlp(deployer, nonce))[12:]`.
   Karena nonce selalu naik, setiap deploy menghasilkan alamat yang berbeda dan bisa
   diprediksi sebelum deploy (lihat [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md#alamat-kontrak-bisa-diprediksi-sebelum-deploy)).

## Aturan yang terbukti di anvil

| Percobaan | Hasil |
| --- | --- |
| Akun #0 kirim ETH 3 kali ke akun #1 | nonce pengirim 59 → 62, nonce penerima tetap **0** |
| Kirim lagi dengan `--nonce 1` (sudah terpakai) | `error code -32003: nonce too low` |
| Raw tx yang sama di-`publish` dua kali | pertama sukses, kedua `nonce too low` (replay ditolak) |
| Deploy init code yang langsung `REVERT` (`0x60006000fd`) | status `0x0`, gas tetap terpakai, nonce tetap naik 65 → 66 |
| Kirim tx dengan nonce 64 saat nonce akun 63 | tx tidak masuk blok; `eth_getTransactionCount … pending` tetap `0x3f` (63) |
| Lalu kirim tx nonce 63 | keduanya masuk blok, nonce jadi **65** |
| Automine dimatikan, kirim nonce 69 (2 gwei), lalu nonce 69 lagi (3 gwei) | tx kedua **menggantikan** tx pertama; blok `0x45` hanya berisi tx kedua (`value` 2) |
| Kirim nonce 69 ketiga kalinya dengan fee yang sama (3 gwei) | `replacement transaction underpriced` |

Pelajaran dari baris terakhir: **satu nonce = satu slot transaksi.** Selama transaksi belum
masuk blok, slot itu bisa ditimpa dengan transaksi lain yang fee-nya lebih tinggi. Wallet
memakai cara ini untuk fitur "speed up" (transaksi sama, fee lebih tinggi) dan "cancel"
(kirim 0 ETH ke diri sendiri dengan nonce yang sama, fee lebih tinggi).

Perintah yang dipakai:

```bash
cast nonce $A -r $RPC                                   # nonce akun (latest)
cast rpc -r $RPC eth_getTransactionCount $A pending     # nonce termasuk tx di mempool
cast send ... --nonce 1                                 # paksa nonce tertentu
RAW=$(cast mktx ...); cast publish $RAW; cast publish $RAW   # uji replay
cast rpc evm_setAutomine false                          # tahan tx di mempool
```

## Nonce kontrak

Kontrak tidak mengirim transaksi, jadi nonce-nya hanya naik saat ia membuat kontrak lain
dengan opcode `CREATE`. Percobaan dengan factory dari bytecode mentah
`600060006000f000` (`CREATE(0,0,0)` lalu `STOP`):

| Langkah | Hasil |
| --- | --- |
| Factory baru di-deploy | nonce factory = **1**, bukan 0 (EIP-161) |
| Factory dipanggil 2 kali (2× `CREATE`) | nonce factory = **3** |
| `cast compute-address <factory> --nonce 1` dan `--nonce 2` | kedua alamat itu sekarang ada, masing-masing dengan nonce **1** |

Akun yang dibuat lewat `CREATE` juga mulai dari nonce 1. Itu sebabnya alamat hasil prediksi
bisa dicek: nonce 1 berarti akun itu sudah dibuat, nonce 0 berarti belum.

`CREATE2` juga menaikkan nonce factory, tetapi alamatnya dihitung dari `salt` dan hash
bytecode, bukan dari nonce.

## Data mainnet Ethereum (2026-10-03)

| Akun | nonce | Artinya |
| --- | --- | --- |
| Vitalik `0xd8dA…6045` | **5966** | sudah mengirim 5.966 transaksi |
| USDC `0xA0b8…eB48` | **1** | kontrak yang belum pernah membuat kontrak lain |
| `0x…dEaD` | **0** | tidak pernah mengirim apa pun; private key-nya tidak diketahui siapa pun |

## Hal yang sering membingungkan

- **Nonce bukan jumlah transaksi masuk.** Menerima ETH tidak mengubah nonce.
- **Transaksi gagal tetap memakai nonce.** Revert atau out-of-gas tetap masuk blok, jadi
  nonce tetap naik dan gas tetap dibayar.
- **Satu nonce yang macet menahan semua nonce sesudahnya.** Kalau tx nonce 10 tertahan karena
  fee terlalu rendah, tx nonce 11, 12, … ikut menunggu. Solusinya: ganti tx nonce 10 dengan
  fee lebih tinggi.
- **Nonce di blok (PoW) itu hal lain.** Header blok juga punya field `nonce`, tapi sejak The
  Merge nilainya selalu 0. Catatan ini membahas nonce akun.

---

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) · ← [EVM 02.01 - EOA dan Contract Account](EVM%2002.01%20-%20EOA%20dan%20Contract%20Account.md)
