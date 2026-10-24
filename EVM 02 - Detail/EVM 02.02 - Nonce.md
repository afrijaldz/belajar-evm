---
title: EVM 02.02 - Nonce
tags: [learning, evm, account, nonce]
updated: 2026-10-24
---

# Nonce

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) · ← [EVM 02.01 - EOA dan Contract Account](EVM%2002.01%20-%20EOA%20dan%20Contract%20Account.md)

Hasil percobaan untuk materi ini ada di [EVM 02.02 - Hasil Praktik](../Hasil%20Praktik/EVM%2002.02%20-%20Hasil%20Praktik.md).

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

## Aturan nonce di node

| Situasi | Yang terjadi |
| --- | --- |
| Akun mengirim transaksi | nonce pengirim naik 1 per transaksi; nonce penerima tidak berubah |
| Tx dengan nonce yang sudah terpakai | ditolak: `nonce too low` |
| Raw tx yang sama dikirim dua kali | yang pertama masuk, yang kedua `nonce too low` (replay ditolak) |
| Tx yang revert (termasuk deploy yang init code-nya langsung `REVERT`) | masuk blok dengan status `0x0`, gas tetap terpakai, nonce tetap naik |
| Tx dengan nonce yang melompat (ada celah) | tertahan di mempool, tidak masuk blok; `eth_getTransactionCount … pending` belum berubah |
| Celahnya diisi | tx pengisi celah dan tx yang tertahan masuk blok berurutan |
| Tx kedua dengan nonce sama dan fee lebih tinggi, sebelum tx pertama masuk blok | tx kedua **menggantikan** tx pertama; blok hanya berisi tx kedua |
| Tx dengan nonce sama dan fee yang sama dengan tx yang sedang menunggu | ditolak: `replacement transaction underpriced` |

Dua baris terakhir menunjukkan bahwa **satu nonce = satu slot transaksi.** Selama transaksi belum
masuk blok, slot itu bisa ditimpa dengan transaksi lain yang fee-nya lebih tinggi. Wallet
memakai cara ini untuk fitur "speed up" (transaksi sama, fee lebih tinggi) dan "cancel"
(kirim 0 ETH ke diri sendiri dengan nonce yang sama, fee lebih tinggi).

Perintah untuk mencobanya:

```bash
cast nonce $A -r $RPC                                   # nonce akun (latest)
cast rpc -r $RPC eth_getTransactionCount $A pending     # nonce termasuk tx di mempool
cast send ... --nonce 1                                 # paksa nonce tertentu
RAW=$(cast mktx ...); cast publish $RAW; cast publish $RAW   # uji replay
cast rpc evm_setAutomine false                          # tahan tx di mempool
cast rpc evm_mine                                       # tambang satu blok
```

## Nonce kontrak

Kontrak tidak mengirim transaksi, jadi nonce-nya hanya naik saat ia membuat kontrak lain
dengan opcode `CREATE`.

- Kontrak yang baru di-deploy langsung punya nonce **1**, bukan 0 (EIP-161).
- Setiap `CREATE` menaikkan nonce kontrak pembuatnya 1. Factory yang sudah memanggil `CREATE`
  dua kali punya nonce 3.
- Kontrak pertama yang dibuat factory ada di `cast compute-address <factory> --nonce 1`, yang
  kedua di `--nonce 2`, dan seterusnya.

Akun yang dibuat lewat `CREATE` juga mulai dari nonce 1. Itu sebabnya alamat hasil prediksi
bisa dicek: nonce 1 berarti akun itu sudah dibuat, nonce 0 berarti belum.

`CREATE2` juga menaikkan nonce factory, tetapi alamatnya dihitung dari `salt` dan hash
bytecode, bukan dari nonce.

## Coba sendiri

Jalankan `anvil`, lalu:

1. Kirim ETH beberapa kali dari akun #0 ke akun #1. Cek nonce keduanya.
2. Kirim lagi dengan `--nonce` yang sudah terpakai.
3. Buat raw tx dengan `cast mktx`, lalu `cast publish` dua kali.
4. Deploy init code yang langsung revert (`0x60006000fd`) dan cek nonce sebelum dan sesudahnya.
5. Kirim tx dengan nonce yang melompat 1, cek nonce `pending`, lalu isi celahnya.
6. Matikan automine, kirim dua tx dengan nonce sama (fee kedua lebih tinggi), lalu tambang blok
   dan lihat isinya. Coba juga tx ketiga dengan fee yang sama.
7. Deploy factory dari bytecode mentah `600060006000f000` (`CREATE(0,0,0)` lalu `STOP`),
   panggil dua kali, lalu cek nonce factory dan alamat hasil `cast compute-address`.
8. Bandingkan nonce beberapa akun di mainnet: wallet aktif, kontrak token, dan `0x…dEaD`.

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
