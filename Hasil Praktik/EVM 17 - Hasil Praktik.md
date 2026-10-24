---
title: EVM 17 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 17 - Hasil Praktik

Hasil percobaan untuk materi [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md). Dijalankan 2026-09-24: proxy sendiri diuji
dengan Foundry (9 test), lalu proxy nyata dibedah di BSC (Safe `0xceba…`) dan Ethereum (USDC). Project:
`~/Documents/riset/evm/proxy` — `src/Proxies.sol` (`NaiveProxy`, `Eip1967Proxy`, `UupsProxy`),
`src/Counters.sol` (versi logika), `test/Proxy.t.sol`. Jalankan: `forge test -vv`.

## `delegatecall` lewat proxy

| | Lewat proxy | Kontrak logika langsung |
| --- | ---: | ---: |
| `count` setelah 2× `increment` | **2** | **0** |
| Code size | 542 byte | 1.227 byte |

## Bug 1: tabrakan storage (`NaiveProxy`)

| | |
| --- | --- |
| `implementation` sebelum `increment()` | `0x5615…b72f` |
| sesudah | `0x5615…b730` (**+1**) |
| Panggil `version()` | `ok = true`, **0 byte** |

Dengan `Eip1967Proxy`: 5× `increment` → slot implementation tetap utuh.

## Bug 2: urutan variabel saat upgrade

| Upgrade | Hasil |
| --- | --- |
| V2 benar | `count` 42 → 43, `owner` tetap alice |
| V2 salah (`owner` dan `count` ditukar) | `owner` = `0x…002A` (angka 42), `count` = 1,75 × 10⁴⁸ (alamat alice + flag) |

## Bug 3: initializer

| | Hasil |
| --- | --- |
| `initialize` dengan flag `initialized` | penyerang → `AlreadyInitialized` |
| `initialize` tanpa pengaman | penyerang memanggil lagi → jadi owner, `setCount(666)` berhasil |

## UUPS

Upgrade ke V2 yang tidak punya `upgradeTo` → berhasil (`count` +2), lalu upgrade berikutnya revert.

## Safe `0xceba60280fb0ecd9a5a26a1552b90944770a4a0e` (BSC, 171 byte) — Safe burn CAKE

`cast disassemble`: `SLOAD` slot 0 → kalau selector `0xa619486e` (`masterCopy()`) kembalikan slot 0 →
selain itu `DELEGATECALL`.

| | |
| --- | --- |
| Slot 0 / `masterCopy()` | `0x3E5c63644E683549055b9Be8653de26E0B4CD36E` (singleton, 23.800 byte) |
| `VERSION()` | 1.3.0 |
| Threshold / owners | **3 dari 7** |

## USDC `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` (Ethereum, 2.186 byte)

| | |
| --- | --- |
| Logika (slot ZeppelinOS `0x7050…f8c3`) | `0x43506849d7c04f9138d1a2050bbf3a0c054402dd` (23.464 byte) |
| **Admin** (slot `0x10d6…390b`) | `0x807a96288a1a408dbc13de2b1d087d10356395d2` — **EOA, 0 byte code** |
| Slot EIP-1967 | **kosong** |

## Token riset: bisa di-upgrade?

| Token | Code | Slot EIP-1967 | `DELEGATECALL` | Kesimpulan |
| --- | ---: | --- | --- | --- |
| UNI | 12.567 byte | kosong | tidak | **kode permanen** |
| ETHFI | 7.889 byte | kosong | tidak | **kode permanen** |
| CAKE | 7.285 byte | kosong | tidak | **kode permanen** |
| USDC | 2.186 byte | kosong | **ya** | proxy, admin EOA |

Untuk riset tokenomics: batas mint 2%/th UNI ([EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) dan `burn()` ETHFI
dijamin **kode**, tidak bisa diganti diam-diam. CAKE token sendiri permanen, tapi emisinya tetap diatur
parameter MasterChef yang dipegang Timelock ([EVM 04 - ABI](../EVM%2004%20-%20ABI.md)).

## Jebakan yang ditemui

- `vm.prank(alice); x.upgradeTo(address(new V2()))` — prank terpakai oleh `new`, bukan `upgradeTo`.
- zsh tidak memecah `$t` (lagi) saat membuat loop multi-kolom; diganti Python.
