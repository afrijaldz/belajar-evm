---
title: EVM 07 - Dasar Solidity
tags: [learning, evm, solidity, foundry]
updated: 2026-10-03
---

# Dasar Solidity

Catatan dari Sesi 7 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24 — sesi pertama Fase 2. Semua konsep dibuktikan
lewat test Foundry (Solidity ^0.8.24, forge 1.7.1). Lanjutan dari [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md).

Project: `~/Documents/riset/evm/solidity-basics` — `src/` (Types, Packing, Locations, Vault,
Exercise) dan `test/`. Jalankan: `forge test -vv`.

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

| Kontrak | Urutan | Slot terpakai | Gas `setAll` pertama |
| --- | --- | ---: | ---: |
| `Unpacked` | `uint128 a; uint256 b; uint128 c;` | 3 | 68.316 |
| `Packed` | `uint128 a; uint128 c; uint256 b;` | 2 | 46.466 |

**Hemat 21.850 gas** hanya dari urutan deklarasi (≈ satu slot baru: 20.000 + 2.100, lihat
[EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)). Slot 0 `Packed` = `…0003` (c, byte 16–31) + `…0001` (a, byte 0–15).

`forge inspect <Kontrak> storage-layout` menampilkan slot dan offset tiap variabel.

Contoh nyata: UNI menyimpan saldo sebagai `uint96` agar checkpoint suara (`uint32` + `uint96`)
muat satu slot — konsekuensinya error "amount exceeds 96 bits" (lihat
[EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)).

## 3. storage vs memory vs calldata

| Lokasi | Sifat | Hasil test |
| --- | --- | --- |
| `memory` | **salinan** sementara | `closeWithMemory` → `active` tetap `true`, perubahan hilang |
| `storage` | **penunjuk** ke slot asli | `closeWithStorage` → `active = false`, tersimpan |
| `calldata` | baca langsung dari input, read-only | **58.362** gas vs 85.794 (`memory`) untuk array 100 elemen, −32% |

**Tanda bug:** compiler memberi warning *"Function state mutability can be restricted to view"*
di `closeWithMemory` — fungsi bernama "close" ternyata tidak menulis apa pun. Warning seperti ini
di fungsi yang seharusnya mengubah state = hampir pasti bug `memory`/`storage`.

Aturan praktis: parameter array/string di fungsi `external` → `calldata`. Mengubah struct di
storage → `storage`.

## 4. Visibility

| | Dipanggil dari luar | Kontrak ini | Kontrak turunan |
| --- | --- | --- | --- |
| `public` | ✅ (variabel dapat getter otomatis) | ✅ | ✅ |
| `external` | ✅ | hanya via `this.f()` | ✅ (dari luar) |
| `internal` | ❌ | ✅ | ✅ |
| `private` | ❌ | ✅ | ❌ (`Undeclared identifier` saat compile) |

**`private` tidak rahasia.** `secretCode` terbaca dari slot 1 dengan `vm.load` (di mainnet:
`cast storage`). `private` cuma menyembunyikan dari kontrak lain. Jangan simpan password atau
kunci di kontrak.

## 5. Modifier dan custom error

| | `require(…, "string")` | custom error |
| --- | ---: | ---: |
| Revert data | **100 byte** | **36 byte** |
| Gas panggilan gagal | 8.152 | **3.668** |
| Bisa membawa data | tidak | ya (`NotOwner(address)`) |

- **Modifier dijalankan kiri ke kanan.** `setCode` punya `onlyOwner whenNotPaused`: saat paused,
  orang asing dapat `NotOwner`, bukan `Paused`, karena `onlyOwner` dicek duluan.
- `_;` = tempat badan fungsi disisipkan.

Tiga jenis revert data yang sudah ditemui:

| Jenis | Selector | Contoh |
| --- | --- | --- |
| `Error(string)` | `0x08c379a0` | `require` dengan pesan ([EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)) |
| `Panic(uint256)` | `0x4e487b71` | overflow, bagi nol, enum |
| Custom error | keccak nama | `InvalidNonce()` Firepit, `NotOwner(address)` |

## 6. Menerima ETH: `payable`, `receive()`, `fallback()`

Kontrak menolak ETH kecuali ETH itu masuk lewat fungsi `payable`, `receive()` (calldata
kosong), atau `fallback()` yang `payable` (selector tidak cocok). Kode: `src/Payable.sol`,
`test/Payable.t.sol`.

### Penjelasan detail

1. [EVM 07.01 - Payable dan receive](EVM%2007%20-%20Detail/EVM%2007.01%20-%20Payable%20dan%20receive.md) — Kapan `receive()`/`fallback()` jalan, pemeriksaan `CALLVALUE` di bytecode, stipend 2.300 gas `transfer`/`send`, ETH paksa lewat `selfdestruct`
2. [EVM 07.02 - Fallback](EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md) — Kapan `fallback()` jalan, dua bentuk penulisan, posisinya di dispatcher bytecode, dan pemakaiannya di proxy (contoh USDC)

## Latihan: `src/Exercise.sol`

Kontrak `DepositBank` dengan 3 masalah. Test di `test/Exercise.t.sol` awalnya **gagal semua**
(sudah diverifikasi bisa diselesaikan).

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
