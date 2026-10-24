---
title: EVM 07.01 - Payable dan receive
tags: [learning, evm, solidity, payable]
updated: 2026-10-24
---

# Payable dan receive()

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · [EVM 07.02 - Fallback](EVM%2007.02%20-%20Fallback.md) →

Hasil percobaan untuk materi ini ada di [EVM 07.01 - Hasil Praktik](../Hasil%20Praktik/EVM%2007.01%20-%20Hasil%20Praktik.md).

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

## Akibat alur ini

| Panggilan | Yang terjadi |
| --- | --- |
| Fungsi `payable` + ETH | sukses, `msg.value` terisi, saldo kontrak bertambah |
| Fungsi non-payable + ETH | **revert**, revert data **0 byte** (tidak ada pesan error) |
| Fungsi non-payable tanpa ETH | sukses |
| ETH polos ke kontrak tanpa `receive()`/`fallback()` | **gagal**, saldo kontrak tidak berubah |
| ETH polos ke kontrak yang punya `receive()` | `receive()` jalan |
| Calldata kosong dan **0 ETH** | `receive()` tetap jalan |
| Fungsi yang tidak ada + ETH | `fallback()` jalan |
| ETH dengan calldata 1 byte (mis. `0x01`) | `fallback()` jalan, bukan `receive()` |
| Fungsi yang ada | fungsi itu jalan, `receive()`/`fallback()` tidak disentuh |
| `fallback()` non-payable, tanpa ETH | sukses |
| `fallback()` non-payable, dengan ETH | **revert** (calldata kosong maupun berisi) |

`receive()` dipilih berdasarkan **calldata kosong**, bukan berdasarkan ada atau tidaknya ETH.

## Isi bytecode: pemeriksaan `CALLVALUE`

Kata `payable` tidak menambahkan kode. Kode tambahan justru muncul di fungsi
**non-payable**: compiler menyisipkan pemeriksaan "ETH harus 0".

Contoh disassembly kontrak kecil `OnlyPayableFunction` dengan fungsi `pay()` (payable),
`notPayable()`, dan variabel publik `lastValue`, tanpa `receive()`/`fallback()`
(`forge inspect … deployedBytecode | cast disassemble`):

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

Akibatnya:

| `receive()` penerima | `transfer` | `send` | `call` |
| --- | --- | --- | --- |
| hanya `emit` (muat di 2.300 gas) | sukses | `true` | sukses |
| menulis storage (mis. `count++`) | **revert** | **`false`** | sukses |

Menulis slot storage baru butuh lebih dari 20.000 gas (lihat [EVM 01.08 - Gas](../EVM%2001%20-%20Detail/EVM%2001.08%20-%20Gas.md)), jauh di
atas 2.300. Jadi `transfer()` dan `send()` gagal mengirim ETH ke kontrak yang `receive()`-nya
melakukan apa pun selain hal sederhana, misalnya ke wallet smart contract. Karena itu
praktik sekarang memakai `call` dan melindunginya dari reentrancy (lihat
[EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md)).

## ETH bisa masuk tanpa melewati receive()

`receive()` **tidak bisa** menolak semua ETH. Kontrak yang memanggil `selfdestruct(target)`
mengirim seluruh saldonya ke `target`, walaupun `target` tidak punya `receive()` maupun
`fallback()`. Saldo target bertambah dan tidak ada kode target yang berjalan.

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
  [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) bagian EIP-7702: kontrak tujuan tanpa `receive()` membuat
  transfer ETH polos revert).

## Coba sendiri

1. Tulis kontrak dengan fungsi `payable`, fungsi non-payable, `receive()`, dan `fallback()`
   dalam beberapa variasi (tanpa keduanya, hanya `receive()`, `fallback()` non-payable).
2. Uji setiap baris tabel "Akibat alur ini" dengan test Foundry. Kirim ETH polos dengan
   `address(target).call{value: 1 ether}("")`.
3. Lihat pemeriksaan `CALLVALUE` di bytecode dengan `forge inspect … deployedBytecode | cast disassemble`.
4. Buat dua penerima (`receive()` hanya `emit` dan `receive()` yang menulis storage), lalu
   kirim ETH ke keduanya dengan `transfer`, `send`, dan `call`.
5. Kirim ETH lewat `selfdestruct` ke kontrak tanpa `receive()`, lalu cek saldonya.

```bash
forge test --match-path test/Payable.t.sol -vv
```

---

Bagian dari [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md) · [EVM 07.02 - Fallback](EVM%2007.02%20-%20Fallback.md) →
