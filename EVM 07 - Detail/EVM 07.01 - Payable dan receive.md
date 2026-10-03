---
title: EVM 07.01 - Payable dan receive
tags: [learning, evm, solidity, payable]
updated: 2026-10-03
---

# Payable dan receive()

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · [EVM 07.02 - Fallback](EVM%2007.02%20-%20Fallback.md) →

**Secara default, kontrak Solidity menolak ETH.** Ada tiga cara agar kontrak bisa menerima
ETH:

| Pintu | Kapan dijalankan | Contoh |
| --- | --- | --- |
| Fungsi `payable` | tx memanggil fungsi itu (selector cocok) dan membawa ETH | `deposit()` di WETH |
| `receive() external payable` | calldata **kosong** | kirim ETH polos dari wallet |
| `fallback() external payable` | selector **tidak cocok** dengan fungsi mana pun, atau calldata kosong tapi tidak ada `receive()` | proxy, kontrak yang menerima data bebas |

- **`payable`** adalah tanda pada fungsi: "fungsi ini boleh menerima ETH". Di dalamnya,
  jumlah ETH bisa dibaca lewat `msg.value`. Fungsi tanpa `payable` langsung revert kalau
  `msg.value > 0`.
- **`receive()`** adalah fungsi khusus tanpa nama, tanpa argumen, dan tanpa return value.
  Fungsi ini wajib `external payable`. Tugasnya hanya satu: menangani ETH yang masuk tanpa
  data.
- **`fallback()`** adalah fungsi khusus untuk panggilan yang tidak cocok dengan fungsi
  mana pun. Fungsi ini boleh `payable` dan boleh tidak.

## Analogi: kantor dengan loket

- **Fungsi `payable`** = loket bertuliskan "terima pembayaran". Kamu menyebut nama loket
  (selector) dan menyerahkan uang.
- **Fungsi non-payable** = loket informasi. Kamu boleh bertanya, tapi uang yang disodorkan
  ditolak.
- **`receive()`** = kotak titipan di depan pintu. Uang tanpa surat apa pun masuk ke sini.
- **`fallback()`** = resepsionis. Ia menangani tamu yang mencari loket yang tidak ada.
- **Kantor tanpa kotak titipan dan tanpa resepsionis yang menerima uang** menolak semua uang
  yang datang tanpa nama loket.

## Alur keputusan

Urutan ini dijalankan oleh kode **dispatcher** yang dibuat compiler di awal bytecode (lihat
[EVM 01.02 - Opcode](../EVM%2001%20-%20Detail/EVM%2001.02%20-%20Opcode.md)):

```
tx masuk ke kontrak
│
├─ calldata kosong?
│   ├─ ya ── ada receive()?  ── ya ──► receive()
│   │                        └─ tidak ─► fallback() (kalau ada), selain itu REVERT
│   │
│   └─ tidak ── 4 byte pertama cocok dengan selector fungsi?
│               ├─ ya ──► fungsi itu
│               └─ tidak ─► fallback() (kalau ada), selain itu REVERT
│
└─ lalu: tx membawa ETH, tapi fungsi tujuan tidak payable? ──► REVERT tanpa pesan
```

## Bukti test Foundry

File: `~/Documents/riset/evm/solidity-basics/src/Payable.sol` dan `test/Payable.t.sol`.

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-path test/Payable.t.sol -vv
```

Hasil: **9 passed, 0 failed** (Solc 0.8.28).

| Test | Hasil |
| --- | --- |
| `pay{value: 1 ether}()` (payable) | sukses, `msg.value` = 1 ETH, saldo kontrak 1 ETH |
| `notPayable()` + 1 ETH | **revert**, revert data **0 byte** (tidak ada pesan error) |
| `notPayable()` tanpa ETH | sukses |
| Kirim ETH polos ke kontrak tanpa `receive()`/`fallback()` | **gagal**, saldo kontrak tetap 0 |
| Kirim ETH polos ke kontrak yang punya `receive()` | `receive()` jalan |
| Panggil dengan calldata kosong dan **0 ETH** | `receive()` tetap jalan |
| Panggil `doesNotExist()` + ETH | `fallback()` jalan |
| Kirim ETH dengan calldata 1 byte (`0x01`) | `fallback()` jalan, bukan `receive()` |
| Panggil `hello()` (fungsi yang ada) | `hello()` jalan, `receive()`/`fallback()` tidak disentuh |
| `fallback()` non-payable: panggilan tanpa ETH | sukses |
| `fallback()` non-payable: dengan ETH | **revert** (calldata kosong maupun berisi) |

Kesimpulan dari baris "calldata kosong dan 0 ETH": `receive()` dipilih berdasarkan
**calldata kosong**, bukan berdasarkan ada atau tidaknya ETH.

## Isi bytecode: pemeriksaan `CALLVALUE`

Kata `payable` tidak menambahkan kode. Kode tambahan justru muncul di fungsi
**non-payable**: compiler menyisipkan pemeriksaan "ETH harus 0".

Disassembly `OnlyPayableFunction` (`forge inspect … deployedBytecode | cast disassemble`):

```
dispatcher:
  0x1b9265b8 pay()        → 0x33   (langsung ke badan fungsi)
  0x273884bd notPayable() → 0x3b
  0x43183834 lastValue()  → 0x4e
  selector tidak cocok    → 0x2f: PUSH0 PUSH0 REVERT   (tidak ada receive/fallback)

0x3b (notPayable):
  CALLVALUE        ; msg.value ke stack
  DUP1
  ISZERO           ; msg.value == 0 ?
  PUSH1 0x45
  JUMPI            ; ya → lanjut ke badan fungsi
  PUSH0 PUSH0
  REVERT           ; tidak → revert(0, 0), tanpa pesan
```

Getter otomatis `lastValue()` juga non-payable, jadi ia mendapat pemeriksaan yang sama di
`0x4e`. Di `pay()`, opcode `CALLVALUE` hanya muncul untuk menyimpan `msg.value` ke storage.

Karena itu, fungsi `payable` sedikit lebih murah: ia melewati beberapa opcode pemeriksaan ini.
Ini bukan alasan untuk membuat semua fungsi `payable`; ETH yang terkirim ke fungsi yang tidak
mengurusnya akan terkunci di kontrak.

## `transfer()`, `send()`, dan `call()`

Ada tiga cara sebuah kontrak mengirim ETH ke kontrak lain. Perbedaannya ada di jumlah gas
yang diteruskan ke `receive()` penerima:

| Cara | Gas untuk penerima | Kalau gagal |
| --- | --- | --- |
| `to.transfer(x)` | **2.300** (stipend) | revert |
| `to.send(x)` | **2.300** | return `false` |
| `to.call{value: x}("")` | semua sisa gas | return `false` |

Hasil test:

| Penerima | `transfer` | `send` | `call` |
| --- | --- | --- | --- |
| `LoggingReceiver` (`receive()` hanya `emit`) | sukses | — | — |
| `CountingReceiver` (`receive()` menulis storage, `count++`) | **revert** | **`false`** | sukses, `count` = 1 |

Menulis slot storage baru butuh lebih dari 20.000 gas (lihat [EVM 01.08 - Gas](../EVM%2001%20-%20Detail/EVM%2001.08%20-%20Gas.md)), jauh di
atas 2.300. Jadi `transfer()` dan `send()` gagal mengirim ETH ke kontrak yang `receive()`-nya
melakukan apa pun selain hal sederhana, misalnya ke wallet smart contract. Karena itu
praktik sekarang memakai `call` dan melindunginya dari reentrancy (lihat
[EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md)).

## ETH bisa masuk tanpa melewati receive()

`receive()` **tidak bisa** menolak semua ETH. Test `test_SelfdestructBypassesReceive`:
kontrak `ForceSender` memanggil `selfdestruct(target)` ke kontrak yang tidak punya `receive()`
maupun `fallback()`. Hasilnya, saldo target menjadi **1 ETH** dan tidak ada kode target yang
berjalan (`lastValue` tetap 0).

Cara lain yang juga melewati kode penerima: alamat kontrak dijadikan penerima block reward
(`coinbase`), atau ETH dikirim ke alamat kontrak sebelum kontrak itu di-deploy (alamatnya
bisa diprediksi, lihat [EVM 02.02 - Nonce](../EVM%2002%20-%20Detail/EVM%2002.02%20-%20Nonce.md)).

Konsekuensinya: **jangan pakai `address(this).balance` sebagai catatan akuntansi.** Simpan
saldo di variabel sendiri yang hanya bertambah lewat fungsi deposit. Ini juga alasan
`receive()` di vault Sesi 12 sengaja revert `UseDepositETH` (lihat
[EVM 12 - Mini Project Vault](../EVM%2012%20-%20Mini%20Project%20Vault.md)).

## Hal yang sering membingungkan

- **`payable` juga dipakai untuk alamat.** `address payable` adalah tipe alamat yang punya
  method `.transfer()` dan `.send()`. Ini berbeda dari fungsi `payable`, walaupun katanya
  sama.
- **Revert karena ETH ke fungsi non-payable tidak punya pesan.** Kalau tx gagal tanpa alasan
  dan tx itu membawa ETH, cek dulu apakah fungsinya `payable`.
- **Constructor juga bisa `payable`.** Tanpa `payable`, deploy kontrak dengan `--value` akan
  revert.
- **EOA tidak punya `receive()`.** EOA selalu menerima ETH karena tidak punya kode. Kecuali
  EOA itu sedang didelegasikan lewat EIP-7702; maka berlaku aturan kontrak tujuan (lihat
  [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) bagian EIP-7702: Counter tanpa `receive()` membuat transfer
  ETH polos revert).

---

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · [EVM 07.02 - Fallback](EVM%2007.02%20-%20Fallback.md) →
