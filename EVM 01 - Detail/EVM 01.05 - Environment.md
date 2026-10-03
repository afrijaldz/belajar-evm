---
title: EVM 01.05 - Environment
tags: [learning, evm, environment]
updated: 2026-10-03
---

# Environment

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.04 - Memory dan Calldata](EVM%2001.04%20-%20Memory%20dan%20Calldata.md) · [EVM 01.06 - Storage](EVM%2001.06%20-%20Storage.md) →

Informasi tentang **situasi saat kode dijalankan**, yang disediakan EVM
dan hanya bisa dibaca: siapa yang memanggil, berapa ETH yang dikirim, blok ke berapa, di chain
mana. Kontrak tidak bisa mengubahnya, dan tidak bisa mengambil data dari luar chain (harga
dari internet, jam di komputer); kalau perlu, data itu harus dibawa masuk lewat calldata atau
oracle. Analoginya: variabel environment saat menjalankan program (`$USER`, `$PWD`, jam sistem).

| Lingkup | Opcode | Di Solidity | Isi |
| --- | --- | --- | --- |
| Call ini | `CALLER` | `msg.sender` | alamat yang **langsung** memanggil (EOA atau kontrak) |
| | `CALLVALUE` | `msg.value` | wei yang dikirim bersama call |
| | `ADDRESS` | `address(this)` | alamat kontrak yang sedang jalan |
| | `GAS` | `gasleft()` | sisa gas |
| Transaksi | `ORIGIN` | `tx.origin` | EOA yang menandatangani transaksi (selalu EOA) |
| | `GASPRICE` | `tx.gasprice` | harga gas efektif |
| Blok | `NUMBER` | `block.number` | nomor blok |
| | `TIMESTAMP` | `block.timestamp` | waktu blok (detik Unix), diatur validator |
| | `CHAINID` | `block.chainid` | 1 Ethereum, 56 BSC, 31337 anvil |
| | `BASEFEE` | `block.basefee` | base fee EIP-1559 |
| | `COINBASE` | `block.coinbase` | penerima fee blok |
| | `PREVRANDAO` | `block.prevrandao` | angka acak dari beacon chain (bisa ditebak validator) |
| | `GASLIMIT`, `BLOCKHASH` | `block.gaslimit`, `blockhash(n)` | batas gas blok; hash 256 blok terakhir |
| Akun lain | `BALANCE`, `SELFBALANCE`, `EXTCODESIZE`, `EXTCODEHASH` | `addr.balance`, `addr.code.length` | baca state akun mana pun |

`CALLER` vs `ORIGIN` paling sering bikin bingung. Kalau wallet memanggil kontrak B, lalu B
memanggil A, maka di dalam A: `CALLER` = **B**, `ORIGIN` = **wallet**. Karena itu cek akses
harus pakai `msg.sender`, bukan `tx.origin` (lihat [EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md)).

```text
wallet 0xf39F…  ──tx──►  kontrak B  ──CALL──►  kontrak A
                                                 CALLER = B
                                                 ORIGIN = 0xf39F… (wallet)
```

Bukti (2026-10-03, anvil): kontrak A menyimpan `CALLER`, `CALLVALUE`, `ORIGIN`, `NUMBER`,
`CHAINID` ke storage.

| | Dipanggil langsung dari wallet + 7 wei | Dipanggil lewat kontrak B |
| --- | --- | --- |
| `CALLER` | `0xf39F…2266` (wallet) | `0x84ea…7feb` (**B**) |
| `CALLVALUE` | 7 | 0 |
| `ORIGIN` | `0xf39F…2266` | `0xf39F…2266` (tetap wallet) |
| `NUMBER` | 34 | 35 |
| `CHAINID` | `0x7a69` = 31337 | 31337 |

Jebakan yang ketemu saat tes: `cast send` ke B tanpa `--gas-limit` memperkirakan gas terlalu
kecil, call ke A gagal `OutOfGas`, tapi transaksinya tetap **sukses**, karena opcode `CALL`
tidak meneruskan kegagalan; pemanggil harus mengecek hasilnya sendiri (Solidity melakukannya
otomatis untuk pemanggilan fungsi biasa, tidak untuk `.call`).

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.04 - Memory dan Calldata](EVM%2001.04%20-%20Memory%20dan%20Calldata.md) · [EVM 01.06 - Storage](EVM%2001.06%20-%20Storage.md) →
