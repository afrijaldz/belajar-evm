---
title: EVM 07.02 - Fallback
tags: [learning, evm, solidity, fallback, proxy]
updated: 2026-10-24
---

# fallback()

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · ← [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md)

Hasil percobaan untuk materi ini ada di [EVM 07.02 - Hasil Praktik](../Hasil%20Praktik/EVM%2007.02%20-%20Hasil%20Praktik.md).

**`fallback()` adalah fungsi cadangan kontrak.** Fungsi ini berjalan kalau tidak ada fungsi
lain yang cocok dengan panggilan. Tanpa `fallback()`, panggilan seperti itu langsung revert.

`fallback()` berjalan dalam dua keadaan:

1. **4 byte pertama calldata tidak cocok dengan selector fungsi mana pun.** Ini juga
   termasuk calldata yang lebih pendek dari 4 byte.
2. **Calldata kosong, tetapi kontrak tidak punya `receive()`.**

## Analogi: resepsionis kantor

Kantor punya loket-loket bernama (fungsi). Tamu datang dan menyebut nama loket (selector).

- Nama loket ada → tamu diantar ke loket itu.
- Nama loket tidak ada → tamu diserahkan ke **resepsionis** (`fallback()`).
- Kantor tanpa resepsionis → tamu ditolak di pintu (revert).

Resepsionis bebas memilih apa yang ia lakukan: menolak dengan sopan (revert dengan pesan),
mencatat tamu (menulis storage), atau **meneruskan tamu ke kantor lain** (proxy).

## Bentuk penulisan

```solidity
// Bentuk 1: tanpa input dan output. Calldata tetap bisa dibaca lewat msg.data.
fallback() external { ... }
fallback() external payable { ... }      // juga menerima ETH

// Bentuk 2: menerima calldata mentah dan mengembalikan bytes mentah.
fallback(bytes calldata input) external returns (bytes memory output) { ... }
```

Aturannya:

- Wajib `external`. Tidak punya nama dan tidak dipanggil dengan nama.
- Boleh `payable` atau tidak. Kalau tidak `payable`, panggilan yang membawa ETH revert.
- Satu kontrak hanya boleh punya satu `fallback()`.
- Di Solidity sebelum 0.6, fallback ditulis sebagai fungsi tanpa nama: `function() external`.
  Bentuk ini masih sering terlihat di kontrak lama.

Pada bentuk 2, output **tidak di-ABI-encode**. Bytes yang dikembalikan langsung menjadi
return data. Ini yang membuat proxy bisa meneruskan return value apa pun tanpa tahu tipenya.

## Apa yang dilihat `fallback()`

| Panggilan | Yang terjadi |
| --- | --- |
| ETH tanpa calldata ke kontrak yang hanya punya `fallback() payable` | `fallback()` jalan, `msg.data.length` 0, `msg.value` terisi |
| Calldata pendek (mis. 2 byte `0xabcd`) | `fallback()` jalan, `msg.data.length` = 2 |
| Fungsi yang tidak ada, mis. `transfer(address,uint256)` | `fallback()` jalan dan melihat calldata utuh, 68 byte (4 selector + 32 + 32) |
| Bentuk 2 yang mengembalikan input apa adanya (mis. `0xdeadbeef01`) | return data persis sama, tanpa ABI encoding |
| `fallback()` yang revert dengan custom error, dipanggil `withdraw()` | revert `UnknownCall(0x3ccfd60b)` (selector `withdraw()`); fungsi lain yang ada tetap jalan normal |
| Proxy dengan `fallback()` + `delegatecall`, dipanggil `increment()` | yang berubah storage proxy, storage kontrak logika tidak berubah |

Aturan di [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md) juga berlaku: `fallback()` non-payable menolak
semua panggilan yang membawa ETH, dan calldata 1 byte + ETH masuk ke `fallback()`, bukan ke
`receive()`.

Compiler memberi peringatan untuk kontrak yang punya `fallback() payable` tanpa `receive()`:

```
Warning (3628): This contract has a payable fallback function, but no receive ether
function. Consider adding a receive ether function.
```

Peringatan ini muncul karena ETH polos akan masuk ke `fallback()`. Sering itu tidak
disengaja.

## Di dalam bytecode

Dispatcher di awal bytecode membandingkan selector satu per satu (lihat
[EVM 01.02 - Opcode](../EVM%2001%20-%20Detail/EVM%2001.02%20-%20Opcode.md)). Bedanya ada di ujung daftar perbandingan:

| Kontrak | Kalau tidak ada selector yang cocok |
| --- | --- |
| Tanpa `fallback()` | lompat ke `PUSH0 PUSH0 REVERT` |
| Dengan `fallback()` | lompat ke badan `fallback()` |

Contoh: di kontrak kecil yang `fallback()`-nya hanya `hits++`, tujuan lompatan cadangan itu
langsung berisi `SLOAD` untuk membaca `hits`.

Calldata yang lebih pendek dari 4 byte juga lompat ke tempat yang sama
(`CALLDATASIZE … LT … JUMPI` di awal bytecode). Jadi `fallback()` hanyalah tujuan lompatan
cadangan di dispatcher.

## Kegunaan utama: proxy

Proxy adalah kontrak yang hampir tidak punya fungsi sendiri. Semua panggilan jatuh ke
`fallback()`, lalu diteruskan dengan `delegatecall` ke kontrak logika (lihat [EVM 17.01 - delegatecall](../EVM%2017%20-%20Detail/EVM%2017.01%20-%20delegatecall.md)). Kode logika berjalan,
tetapi storage yang dipakai milik proxy (lihat [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md)).

```
user ──"increment()"──► MiniProxy
                         │ selector tidak ada di proxy
                         ▼
                       fallback(input)
                         │ delegatecall(input)
                         ▼
                       CounterLogic.increment()  ── menulis slot 0 MILIK PROXY
```

Contoh nyata: **USDC** di Ethereum mainnet adalah proxy. Bytecode proxy-nya kecil dan hanya
berisi fungsi milik proxy sendiri (`upgradeTo`, `implementation`, `admin`). Selector `symbol()`
(`0x95d89b41`) tidak ada di bytecode proxy, tetapi `cast call <USDC> "symbol()(string)"` tetap
mengembalikan `"USDC"`. Panggilan itu masuk ke `fallback()` proxy dan diteruskan ke kontrak
logika, yang alamatnya disimpan di slot implementation gaya ZeppelinOS milik proxy.

## Kegunaan lain

- **Menolak dengan pesan yang jelas.** `fallback()` bisa revert dengan
  `UnknownCall(selector)`, bukan revert kosong. Ini memudahkan debugging.
- **Menerima ETH dan data sekaligus.** `fallback() payable` menerima ETH yang dikirim bersama
  calldata yang tidak dikenal.
- **Kontrak yang membaca calldata sendiri.** Beberapa kontrak MEV dan router hemat gas
  memakai format calldata sendiri tanpa selector, dan semuanya dibaca di `fallback()`.

## Hal yang sering membingungkan

- **`fallback()` vs `receive()`.** `receive()` hanya untuk calldata kosong. `fallback()`
  untuk selector yang tidak cocok, dan juga untuk calldata kosong kalau tidak ada
  `receive()`.
- **`fallback()` bisa menyembunyikan salah ketik.** Kalau nama fungsi salah ketik di sisi
  pemanggil, panggilan tidak revert. Panggilan itu masuk diam-diam ke `fallback()` dan
  terlihat sukses.
- **Selector bentrok di proxy.** Kalau proxy punya fungsi dengan selector yang sama dengan
  fungsi di kontrak logika, fungsi proxy yang menang, dan fungsi logika tidak pernah
  terpanggil. Transparent proxy dan UUPS ada untuk menangani masalah ini (lihat
  [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md)).

## Coba sendiri

1. Tulis kontrak yang hanya punya `fallback() payable`, lalu kirim ETH tanpa calldata dan
   panggil dengan calldata 2 byte. Cek `msg.data.length` dan `msg.value`.
2. Tulis `fallback()` bentuk 2 yang mengembalikan input apa adanya, lalu bandingkan return
   data dengan input.
3. Tulis `fallback()` yang revert `UnknownCall(bytes4)`, lalu panggil fungsi yang tidak ada.
4. Buat proxy mini: `fallback(bytes calldata input)` yang `delegatecall` ke kontrak counter.
   Panggil `increment()` dua kali, lalu bandingkan storage proxy dan kontrak logika.
5. Cek proxy nyata: cari selector `symbol()` di `cast code <USDC>`, lalu panggil
   `cast call <USDC> "symbol()(string)"` dan baca slot implementation dengan `cast storage`.

```bash
forge test --match-path test/Fallback.t.sol -vv
```

---

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · ← [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md)
