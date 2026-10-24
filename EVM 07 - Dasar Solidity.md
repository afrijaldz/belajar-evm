---
title: EVM 07 - Dasar Solidity
tags: [learning, evm, solidity, foundry]
updated: 2026-10-24
---

# Dasar Solidity

Sesi 7 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md) — sesi pertama Fase 2, lanjutan dari
[EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md). Di sesi ini kamu belajar dasar Solidity yang paling
sering menentukan benar-salahnya kontrak: tipe data dan overflow, storage packing, lokasi data,
visibility, modifier dan custom error, serta cara kontrak menerima ETH. Setiap konsep bisa
dibuktikan dengan test Foundry (Solidity ^0.8.24).

Hasil percobaan untuk materi ini ada di [EVM 07 - Hasil Praktik](Hasil%20Praktik/EVM%2007%20-%20Hasil%20Praktik.md).

## 1. Tipe data dan overflow

| Kasus | Perilaku | Revert data |
| --- | --- | --- |
| `uint8` 200 + 100 | revert | `Panic(0x11)` overflow |
| `uint256` 1 − 2 | revert | `Panic(0x11)` underflow |
| 1 / 0 | revert | `Panic(0x12)` |
| `Status(3)` (enum 0–2) | revert | `Panic(0x21)` |
| 200 + 100 dalam `unchecked` | **44** | — (memutar: 300 mod 256) |
| 7 / 2 | **3** | — (bulat ke bawah) |

- Sejak 0.8 aritmetika **dicek otomatis**. Sebelumnya overflow memutar diam-diam — kontrak lama
  (MasterChefV1, 2020) pakai library `SafeMath`.
- `unchecked` dipakai sengaja untuk hemat gas di tempat yang pasti tidak overflow.
- Tidak ada desimal → token pakai `decimals = 18`.
- Batas: `uint8` 255, `int8` −128…127, `uint256` ≈ 1,16 × 10^77.

## 2. Storage packing

Variabel kecil yang **berurutan** digabung dalam satu slot 32 byte.

| Kontrak | Urutan | Slot terpakai |
| --- | --- | ---: |
| `Unpacked` | `uint128 a; uint256 b; uint128 c;` | 3 |
| `Packed` | `uint128 a; uint128 c; uint256 b;` | 2 |

Hanya dengan mengubah urutan deklarasi, `Packed` menghemat satu slot. Penulisan pertama ke semua
variabel jadi lebih murah sekitar biaya satu slot baru (20.000 + 2.100, lihat
[EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)). Di slot 0 `Packed`, `a` mengisi byte 0–15 dan `c` mengisi byte
16–31 — kalau `a = 1` dan `c = 3`, slot 0 berisi `…0003` + `…0001`.

`forge inspect <Kontrak> storage-layout` menampilkan slot dan offset tiap variabel.

Contoh nyata: UNI menyimpan saldo sebagai `uint96` agar checkpoint suara (`uint32` + `uint96`)
muat satu slot — konsekuensinya error "amount exceeds 96 bits" (lihat
[EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)).

## 3. storage vs memory vs calldata

| Lokasi | Sifat | Akibat |
| --- | --- | --- |
| `memory` | **salinan** sementara | mengubah struct dari storage lewat variabel `memory` → perubahan hilang |
| `storage` | **penunjuk** ke slot asli | mengubah lewat variabel `storage` → tersimpan |
| `calldata` | baca langsung dari input, read-only | lebih murah dari `memory` untuk parameter array, karena tidak perlu disalin |

**Tanda bug:** compiler memberi warning *"Function state mutability can be restricted to view"*
di fungsi yang memakai `memory` padahal bermaksud mengubah state — misalnya fungsi bernama
"close" yang sebenarnya tidak menulis apa pun. Warning seperti ini di fungsi yang seharusnya
mengubah state = hampir pasti bug `memory`/`storage`.

Aturan praktis: parameter array/string di fungsi `external` → `calldata`. Mengubah struct di
storage → `storage`.

## 4. Visibility

| | Dipanggil dari luar | Kontrak ini | Kontrak turunan |
| --- | --- | --- | --- |
| `public` | ✅ (variabel dapat getter otomatis) | ✅ | ✅ |
| `external` | ✅ | hanya via `this.f()` | ✅ (dari luar) |
| `internal` | ❌ | ✅ | ✅ |
| `private` | ❌ | ✅ | ❌ (`Undeclared identifier` saat compile) |

**`private` tidak rahasia.** Variabel `private` tetap bisa dibaca langsung dari slot storage-nya
dengan `vm.load` di test, atau `cast storage` di mainnet. `private` cuma menyembunyikan dari
kontrak lain. Jangan simpan password atau kunci di kontrak.

## 5. Modifier dan custom error

| | `require(…, "string")` | custom error |
| --- | ---: | ---: |
| Revert data (pesan pendek) | **100 byte** | **36 byte** (dengan satu argumen `address`) |
| Gas panggilan gagal | lebih mahal | **jauh lebih murah** |
| Bisa membawa data | tidak | ya (`NotOwner(address)`) |

- **Modifier dijalankan kiri ke kanan.** Fungsi dengan `onlyOwner whenNotPaused`: saat paused,
  orang asing dapat `NotOwner`, bukan `Paused`, karena `onlyOwner` dicek duluan.
- `_;` = tempat badan fungsi disisipkan.

Tiga jenis revert data:

| Jenis | Selector | Contoh |
| --- | --- | --- |
| `Error(string)` | `0x08c379a0` | `require` dengan pesan ([EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)) |
| `Panic(uint256)` | `0x4e487b71` | overflow, bagi nol, enum |
| Custom error | keccak nama | `InvalidNonce()` Firepit, `NotOwner(address)` |

## 6. Menerima ETH: `payable`, `receive()`, `fallback()`

Kontrak menolak ETH kecuali ETH itu masuk lewat fungsi `payable`, `receive()` (calldata
kosong), atau `fallback()` yang `payable` (selector tidak cocok).

### Penjelasan detail

1. [EVM 07.01 - Payable dan receive](EVM%2007%20-%20Detail/EVM%2007.01%20-%20Payable%20dan%20receive.md) — Kapan `receive()`/`fallback()` jalan, pemeriksaan `CALLVALUE` di bytecode, stipend 2.300 gas `transfer`/`send`, ETH paksa lewat `selfdestruct`
2. [EVM 07.02 - Fallback](EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md) — Kapan `fallback()` jalan, dua bentuk penulisan, posisinya di dispatcher bytecode, dan pemakaiannya di proxy (contoh USDC)

## Latihan: `src/Exercise.sol`

Kontrak `DepositBank` dengan 3 masalah. Test di `test/Exercise.t.sol` awalnya **gagal semua**;
tugasmu membuatnya lulus.

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-contract ExerciseTest -vv
```

| Task | Masalah | Petunjuk |
| --- | --- | --- |
| 1 | `Deposit` pakai 4 slot | `uint256` selalu satu slot penuh. `address` = 20 byte, `uint64` = 8, `bool` = 1. Mana yang muat bareng? |
| 2 | `withdraw` tidak menyimpan `withdrawn = true` | Lihat bagian 3 |
| 3 | Siapa pun bisa withdraw | `error NotDepositOwner(address caller)` + modifier `onlyDepositOwner(uint256 id)` |

Status: ☐ belum dikerjakan

## Jebakan

- Mengimpor file di luar project gagal (`Source … not found`) — Foundry mencari relatif ke root
  project.

## Coba sendiri

1. Buat project dengan `forge init`.
2. Tulis test untuk setiap baris tabel overflow, pakai `vm.expectRevert` untuk kasus yang revert.
3. Buat kontrak `Unpacked` dan `Packed`, bandingkan `forge inspect <Kontrak> storage-layout` dan gas
   penulisan pertamanya.
4. Tulis fungsi yang mengubah struct lewat `memory` dan lewat `storage`, lalu bandingkan hasilnya.
5. Simpan variabel `private`, lalu baca slot-nya dengan `vm.load`.
6. Bandingkan revert data dan gas `require` dengan pesan vs custom error.

```bash
forge test -vv
forge inspect <Kontrak> storage-layout
```
