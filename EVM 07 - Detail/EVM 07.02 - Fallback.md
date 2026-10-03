---
title: EVM 07.02 - Fallback
tags: [learning, evm, solidity, fallback, proxy]
updated: 2026-10-03
---

# fallback()

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · ← [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md)

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

## Bukti test Foundry

File: `~/Documents/riset/evm/solidity-basics/src/Fallback.sol` dan `test/Fallback.t.sol`.

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-path test/Fallback.t.sol -vv
```

Hasil: **6 passed, 0 failed**.

| Test | Hasil |
| --- | --- |
| Kirim 1 ETH tanpa calldata ke kontrak yang hanya punya `fallback() payable` | `fallback()` jalan, `msg.data.length` 0, `msg.value` 1 ETH |
| Panggil dengan calldata 2 byte (`0xabcd`) | `fallback()` jalan, `msg.data.length` **2** |
| Panggil `transfer(address,uint256)` ke kontrak yang tidak punya fungsi itu | `fallback()` jalan dan melihat calldata utuh, **68 byte** (4 selector + 32 + 32) |
| `EchoFallback` (bentuk 2) dipanggil dengan `0xdeadbeef01` | return data persis `0xdeadbeef01`, tanpa ABI encoding |
| `StrictFallback` dipanggil dengan `withdraw()` | revert `UnknownCall(0x3ccfd60b)` (selector `withdraw()`); fungsi `known()` tetap jalan normal |
| `MiniProxy` dipanggil `increment()` 2 kali | `number` di proxy = **2**, di kontrak logika tetap **0**; slot 0 proxy = 2 |

Hasil test di [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md) juga berlaku: `fallback()` non-payable
menolak semua panggilan yang membawa ETH, dan calldata 1 byte + ETH masuk ke `fallback()`,
bukan ke `receive()`.

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
| Tanpa `fallback()` (`OnlyPayableFunction`) | lompat ke `PUSH0 PUSH0 REVERT` |
| Dengan `fallback()` (`FallbackOnly`) | lompat ke `0x37`, yaitu badan `fallback()` (langsung `SLOAD` untuk `hits++`) |

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

Contoh nyata di Ethereum mainnet (2026-10-03), USDC `0xA0b8…eB48`:

| Cek | Hasil |
| --- | --- |
| Ukuran bytecode proxy | 2.186 byte |
| Selector `symbol()` `0x95d89b41` ada di bytecode proxy? | **tidak** |
| `cast call <USDC> "symbol()(string)"` | `"USDC"` |
| Selector `upgradeTo`, `implementation`, `admin` di bytecode proxy | ada (fungsi milik proxy sendiri) |
| Slot ZeppelinOS implementation `0x7050…f8c3` | `0x43506849d7c04f9138d1a2050bbf3a0c054402dd` |

Fungsi `symbol()` tidak ada di proxy, tetapi panggilan tetap berhasil. Panggilan itu masuk ke
`fallback()` proxy dan diteruskan ke kontrak logika `0x4350…02dd`.

## Kegunaan lain

- **Menolak dengan pesan yang jelas.** `StrictFallback` revert dengan `UnknownCall(selector)`,
  bukan revert kosong. Ini memudahkan debugging.
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

---

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · ← [EVM 07.01 - Payable dan receive](EVM%2007.01%20-%20Payable%20dan%20receive.md)
