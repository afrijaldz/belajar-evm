---
title: EVM 17 - Proxy dan Upgrade
tags: [learning, evm, solidity, proxy, upgrade, security]
updated: 2026-10-03
---

# Proxy dan Upgrade

Catatan dari Sesi 17 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Proxy ditulis sendiri dan diuji dengan Foundry
(9 test), lalu proxy nyata dibedah di BSC (Safe `0xceba…`) dan Ethereum (USDC). Lanjutan dari
[EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md); menjawab proxy yang pertama kali ditemui di [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md).

Project: `~/Documents/riset/evm/proxy` — `src/Proxies.sol` (`NaiveProxy`, `Eip1967Proxy`, `UupsProxy`),
`src/Counters.sol` (versi logika), `test/Proxy.t.sol`. Jalankan: `forge test -vv`.

## `delegatecall`: kode dari kontrak lain, storage milik sendiri

Proxy meneruskan setiap panggilan lewat `delegatecall` ke kontrak logika. Kode logika berjalan di atas
**storage proxy**, dengan `msg.sender` / `msg.value` tetap milik pemanggil asli.

| | Lewat proxy | Kontrak logika langsung |
| --- | ---: | ---: |
| `count` setelah 2× `increment` | **2** | **0** |
| Code size | 542 byte | 1.227 byte |

Constructor logika berjalan di kontrak logika, bukan di proxy → setup lewat **`initialize`**.

### Penjelasan detail

1. [EVM 17.01 - delegatecall](EVM%2017%20-%20Detail/EVM%2017.01%20-%20delegatecall.md) — `call` vs `delegatecall` (storage, `msg.sender`, `address(this)`, ETH), opcode `0xf4`, delegatecall ke alamat tanpa kode, pengambilalihan wallet, insiden Parity

## Bug 1: tabrakan storage

`NaiveProxy` menyimpan `implementation` di slot 0 — tempat `CounterV1` menyimpan `count`.

| | |
| --- | --- |
| `implementation` sebelum `increment()` | `0x5615…b72f` |
| sesudah | `0x5615…b730` (**+1**) |
| Panggil `version()` | `ok = true`, **0 byte** |

**`delegatecall` ke alamat tanpa kode dianggap berhasil.** Proxy rusak tidak revert — diam-diam tidak
melakukan apa pun.

**Perbaikan EIP-1967:** simpan data proxy di slot pseudo-acak.

```
implementation: 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc = keccak256("eip1967.proxy.implementation") − 1
admin:          0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103 = keccak256("eip1967.proxy.admin") − 1
```

5× `increment` → slot implementation tetap utuh.

## Bug 2: urutan variabel diubah saat upgrade

Storage tidak ikut pindah saat upgrade; yang berubah hanya nama yang menunjuk ke slot.

| Upgrade | Hasil |
| --- | --- |
| **V2 benar** — variabel lama di tempat sama, variabel baru **ditambah di akhir** | `count` 42 → 43, `owner` tetap alice |
| **V2 salah** — `owner` dan `count` ditukar | `owner` = `0x…002A` (angka 42), `count` = 1,75 × 10⁴⁸ (alamat alice + flag) |

Aturan: **hanya menambah di akhir**, jangan mengubah urutan/tipe/menghapus. `forge inspect <K> storage-layout`
untuk membandingkan.

## Bug 3: initializer tanpa pengaman

| | Hasil |
| --- | --- |
| `initialize` dengan flag `initialized` | penyerang → `AlreadyInitialized` |
| `initialize` tanpa pengaman | penyerang memanggil lagi → **jadi owner**, `setCount(666)` berhasil |

## Transparent vs UUPS

| | Transparent (`Eip1967Proxy`) | UUPS (`UupsProxy`) |
| --- | --- | --- |
| Fungsi upgrade ada di | proxy (hanya admin) | **kontrak logika** |
| Non-admin memanggil `upgradeTo` | diteruskan ke logika → revert | tergantung logika |
| Risiko khas | admin tidak bisa memakai fungsi logika | **lupa `upgradeTo` di versi baru = beku selamanya** |

Test UUPS: upgrade ke V2 yang tidak punya `upgradeTo` → berhasil (`count` +2), lalu upgrade berikutnya
**revert** — tidak ada lagi jalan untuk upgrade.

## Proxy nyata

### Safe `0xceba60280fb0ecd9a5a26a1552b90944770a4a0e` (BSC, 171 byte) — Safe burn CAKE

`cast disassemble`: `SLOAD` slot 0 → kalau selector `0xa619486e` (`masterCopy()`) kembalikan slot 0 →
selain itu `DELEGATECALL`.

| | |
| --- | --- |
| Slot 0 / `masterCopy()` | `0x3E5c63644E683549055b9Be8653de26E0B4CD36E` (singleton, 23.800 byte) |
| `VERSION()` | 1.3.0 |
| Threshold / owners | **3 dari 7** |

Safe menaruh alamat logika **di slot 0** — seperti `NaiveProxy` yang rusak — tapi **aman dan disengaja**:
kontrak logika Safe juga menyimpan `singleton` di slot 0, jadi keduanya selalu sejajar. Proxy ini tidak
punya fungsi upgrade.

### USDC `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` (Ethereum, 2.186 byte)

| | |
| --- | --- |
| Logika (slot ZeppelinOS `0x7050…f8c3`) | `0x43506849d7c04f9138d1a2050bbf3a0c054402dd` (23.464 byte) |
| **Admin** (slot `0x10d6…390b`) | `0x807a96288a1a408dbc13de2b1d087d10356395d2` — **EOA, 0 byte code** |
| Slot EIP-1967 | **kosong** |

Satu private key bisa mengganti seluruh logika USDC. Dan **cek slot EIP-1967 saja akan salah
menyimpulkan USDC tidak bisa di-upgrade** — USDC memakai slot lama ZeppelinOS. Cek juga `DELEGATECALL` di
bytecode.

### Token riset: bisa di-upgrade?

| Token | Code | Slot EIP-1967 | `DELEGATECALL` | Kesimpulan |
| --- | ---: | --- | --- | --- |
| UNI | 12.567 byte | kosong | tidak | **kode permanen** |
| ETHFI | 7.889 byte | kosong | tidak | **kode permanen** |
| CAKE | 7.285 byte | kosong | tidak | **kode permanen** |
| USDC | 2.186 byte | kosong | **ya** | proxy, admin EOA |

Untuk riset tokenomics: batas mint 2%/th UNI ([EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) dan `burn()` ETHFI
dijamin **kode**, tidak bisa diganti diam-diam. CAKE token sendiri permanen, tapi emisinya tetap diatur
parameter MasterChef yang dipegang Timelock ([EVM 04 - ABI](EVM%2004%20-%20ABI.md)).

## Jebakan

- **`vm.prank` + `new` di dalam argumen**: `vm.prank(alice); x.upgradeTo(address(new V2()))` — prank
  terpakai oleh `new`, bukan `upgradeTo`. Deploy dulu, baru prank.
- zsh tidak memecah `$t` (lagi) — loop multi-kolom pakai Python.
