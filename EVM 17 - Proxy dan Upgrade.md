---
title: EVM 17 - Proxy dan Upgrade
tags: [learning, evm, solidity, proxy, upgrade, security]
updated: 2026-10-24
---

# Proxy dan Upgrade

Sesi 17 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu menulis proxy sendiri, mengujinya dengan Foundry, lalu
membedah proxy nyata di BSC (Safe) dan Ethereum (USDC). Lanjutan dari [EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md);
menjawab proxy yang pertama kali muncul di [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md).

Hasil percobaan untuk materi ini ada di [EVM 17 - Hasil Praktik](Hasil%20Praktik/EVM%2017%20-%20Hasil%20Praktik.md).

## `delegatecall`: kode dari kontrak lain, storage milik sendiri

Proxy meneruskan setiap panggilan lewat `delegatecall` ke kontrak logika. Kode logika berjalan di atas
**storage proxy**, dengan `msg.sender` / `msg.value` tetap milik pemanggil asli. Akibatnya, state hanya
berubah di proxy: `count` yang dinaikkan lewat proxy tidak mengubah `count` di kontrak logika. Proxy-nya
sendiri kecil, karena semua logika ada di kontrak lain.

Constructor logika berjalan di kontrak logika, bukan di proxy → setup lewat **`initialize`**.

### Penjelasan detail

1. [EVM 17.01 - delegatecall](EVM%2017%20-%20Detail/EVM%2017.01%20-%20delegatecall.md) — `call` vs `delegatecall` (storage, `msg.sender`, `address(this)`, ETH), opcode `0xf4`, delegatecall ke alamat tanpa kode, pengambilalihan wallet, insiden Parity

## Bug 1: tabrakan storage

`NaiveProxy` menyimpan `implementation` di slot 0 — tempat `CounterV1` menyimpan `count`. Saat logika
menjalankan `increment()`, yang bertambah adalah slot 0 milik proxy, yaitu alamat `implementation`. Alamat
itu naik 1 dan sekarang menunjuk ke alamat yang tidak punya kode.

**`delegatecall` ke alamat tanpa kode dianggap berhasil.** Panggilan berikutnya (mis. `version()`) mengembalikan
`ok = true` dengan 0 byte data. Proxy rusak tidak revert — diam-diam tidak melakukan apa pun.

**Perbaikan EIP-1967:** simpan data proxy di slot pseudo-acak yang tidak mungkin dipakai variabel biasa.

```
implementation: 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc = keccak256("eip1967.proxy.implementation") − 1
admin:          0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103 = keccak256("eip1967.proxy.admin") − 1
```

Dengan slot ini, berapa kali pun `increment` dipanggil, slot implementation tetap utuh.

## Bug 2: urutan variabel diubah saat upgrade

Storage tidak ikut pindah saat upgrade; yang berubah hanya nama yang menunjuk ke slot.

| Upgrade | Akibat |
| --- | --- |
| **V2 benar** — variabel lama di tempat sama, variabel baru **ditambah di akhir** | nilai lama terbaca benar, `count` lanjut dari angka sebelumnya, `owner` tetap |
| **V2 salah** — `owner` dan `count` ditukar | `owner` membaca angka `count` sebagai alamat, `count` membaca alamat owner (+ flag) sebagai angka raksasa |

Aturan: **hanya menambah di akhir**, jangan mengubah urutan/tipe/menghapus. `forge inspect <K> storage-layout`
untuk membandingkan.

## Bug 3: initializer tanpa pengaman

| | Akibat |
| --- | --- |
| `initialize` dengan flag `initialized` | panggilan kedua revert `AlreadyInitialized` |
| `initialize` tanpa pengaman | penyerang memanggil lagi → **jadi owner**, lalu bisa memanggil fungsi khusus owner (mis. `setCount`) |

## Transparent vs UUPS

| | Transparent (`Eip1967Proxy`) | UUPS (`UupsProxy`) |
| --- | --- | --- |
| Fungsi upgrade ada di | proxy (hanya admin) | **kontrak logika** |
| Non-admin memanggil `upgradeTo` | diteruskan ke logika → revert | tergantung logika |
| Risiko khas | admin tidak bisa memakai fungsi logika | **lupa `upgradeTo` di versi baru = beku selamanya** |

Di UUPS, upgrade ke versi yang tidak punya `upgradeTo` tetap berhasil dan versi itu berjalan normal. Tapi
upgrade berikutnya **revert** — tidak ada lagi jalan untuk upgrade.

## Proxy nyata

### Safe (BSC) — Safe burn CAKE

Proxy Safe sangat kecil. Isinya: `SLOAD` slot 0 → kalau selector `0xa619486e` (`masterCopy()`) kembalikan
slot 0 → selain itu `DELEGATECALL` ke alamat di slot 0.

Safe menaruh alamat logika **di slot 0** — seperti `NaiveProxy` yang rusak — tapi **aman dan disengaja**:
kontrak logika Safe (singleton) juga menyimpan `singleton` di slot 0, jadi keduanya selalu sejajar. Proxy ini
tidak punya fungsi upgrade.

### USDC (Ethereum)

USDC adalah proxy yang memakai **slot lama ZeppelinOS**, bukan slot EIP-1967. Slot admin-nya bisa berisi
sebuah EOA — artinya satu private key bisa mengganti seluruh logika token.

**Jebakan:** cek slot EIP-1967 saja akan salah menyimpulkan USDC tidak bisa di-upgrade. Cek juga
`DELEGATECALL` di bytecode.

### Cara mengecek apakah token bisa di-upgrade

1. Baca slot EIP-1967 implementation dan admin (`cast storage`).
2. Cari opcode `DELEGATECALL` di bytecode (`cast disassemble`).
3. Kalau dua-duanya kosong, kode token **permanen**: aturan di kodenya (mis. batas mint) tidak bisa diganti
   diam-diam. Tapi parameter yang tinggal di kontrak lain (mis. emisi di MasterChef yang dipegang Timelock,
   [EVM 04 - ABI](EVM%2004%20-%20ABI.md)) tetap bisa diubah pemegangnya.
4. Kalau proxy, cari siapa admin-nya: kontrak (multisig/Timelock) atau EOA.

## Jebakan

- **`vm.prank` + `new` di dalam argumen**: `vm.prank(alice); x.upgradeTo(address(new V2()))` — prank
  terpakai oleh `new`, bukan `upgradeTo`. Deploy dulu, baru prank.
- zsh tidak memecah `$t` menjadi beberapa kata seperti bash — loop multi-kolom pakai Python.

## Coba sendiri

1. Tulis `NaiveProxy`, `Eip1967Proxy`, `UupsProxy`, dan beberapa versi `Counter` (V1, V2 benar, V2 salah).
2. Panggil `increment` lewat proxy dan langsung ke logika; bandingkan `count` dan code size.
3. Lewat `NaiveProxy`, catat alamat `implementation` sebelum dan sesudah `increment()`, lalu panggil `version()`.
4. Upgrade ke V2 benar dan V2 salah; baca `owner` dan `count`.
5. Coba panggil `initialize` dua kali, dengan dan tanpa pengaman.
6. UUPS: upgrade ke versi tanpa `upgradeTo`, lalu coba upgrade lagi.
7. Bedah proxy nyata dengan perintah di bawah.

```bash
forge test -vv
forge inspect <Kontrak> storage-layout
cast storage <proxy> 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc   # implementation EIP-1967
cast storage <proxy> 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103   # admin EIP-1967
cast disassemble $(cast code <alamat>) | grep DELEGATECALL
```
