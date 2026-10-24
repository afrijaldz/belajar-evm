---
title: EVM 12 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 12 - Hasil Praktik

Hasil percobaan untuk materi [EVM 12 - Mini Project Vault](../EVM%2012%20-%20Mini%20Project%20Vault.md). Dijalankan 2026-09-24 — penutup
Fase 2. Vault diuji dengan 12 test unit + 2 invariant (256.000 panggilan acak).

Project: `~/Documents/riset/evm/vault-project` — `src/Vault.sol` (benar), `src/NaiveVault.sol`
(sengaja salah), `src/mocks/MockTokens.sol`, `test/Vault.t.sol`, `test/VaultInvariant.t.sol`.

## Pembaruan Sesi 19

Dari [EVM 19 - Testing Lanjutan](../EVM%2019%20-%20Testing%20Lanjutan.md): coverage cabang menemukan pengecekan `token.code.length` di
`_safeCall` tidak pernah tercapai dari `depositToken` (balanceOf revert duluan), dan
`test_TokenWithNoCodeIsRejected` lolos karena revert yang salah. Pengecekan dipindah ke awal
`depositToken`, test diperketat, 5 test cabang + invariant ghost ditambah. Sekarang 17 test, cabang
100%.

## Reentrancy

| | NaiveVault | Vault |
| --- | ---: | ---: |
| ETH vault setelah serangan (awal 10 milik Alice + 1 penyerang) | **0** | 10 |
| ETH penyerang | **11** | 1 (miliknya sendiri) |
| Re-entry | 10 kali | gagal (`InsufficientBalance`) |
| Alice masih tercatat punya | 10 ETH (vault kosong) | 10 ETH (aman) |

## Fee-on-transfer (1%)

Alice dan Bob masing-masing deposit 100: NaiveVault mencatat 200 tapi memegang 198, dan Bob revert
saat withdraw (tinggal 98 untuk klaim 100). Vault mencatat 198 (99 + 99), keduanya berhasil withdraw.

## Token tanpa `bool`

`NaiveVault` revert di `require(IERC20Bool(token).transferFrom(...))`; `Vault` berhasil deposit dan
withdraw lewat `_safeCall`.

## Invariant test

Handler `VaultHandler`: 3 aktor × 4 aset (ETH, token normal, token fee, token ala USDT), 256 run ×
500 panggilan.

### Bug di test sendiri: 20% panggilan revert diam-diam

Run pertama lolos, tapi **~25.000 dari 128.000 panggilan revert**. Penyebab: handler memanggil
`MockToken(asset).approve(...)` untuk semua token, termasuk `NoReturnToken` → jebakan 3 menimpa
**kode test**. Akibatnya deposit token ala USDT tidak pernah terjadi — invariant tidak pernah menguji
skenario itu, tapi tetap "lolos".

Perbaikan: approve lewat low-level call. Hasil: **0 revert dari 256.000 panggilan**. Lalu
`fail_on_revert = true` dipasang di `foundry.toml`.

### Mutation test

Bug fee-on-transfer ditanam kembali (kredit `amount`) di salinan project:

| Invariant | Hasil |
| --- | --- |
| `invariant_VaultIsSolvent` | ❌ **"vault holds less than it owes"** (3,98e12 < 4,02e12) |
| `invariant_UserBalancesAddUpToTotal` | ❌ `TokenCallFailed(feeToken)` — withdraw gagal, tertangkap `fail_on_revert` |

Gagal dalam 3 detik.
