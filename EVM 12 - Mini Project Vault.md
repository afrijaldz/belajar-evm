---
title: EVM 12 - Mini Project Vault
tags: [learning, evm, solidity, foundry, security, invariant]
updated: 2026-09-24
---

# Mini Project Vault

Catatan dari Sesi 12 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24 — **penutup Fase 2**. Vault deposit/withdraw ETH +
ERC-20 yang menggabungkan Sesi 7–11, diuji dengan 12 test unit + 2 invariant (256.000 panggilan
acak). Lanjutan dari [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md).

Project: `~/Documents/riset/evm/vault-project` — `src/Vault.sol` (benar), `src/NaiveVault.sol`
(sengaja salah), `src/mocks/MockTokens.sol`, `test/Vault.t.sol`, `test/VaultInvariant.t.sol`.

> **Pembaruan Sesi 19 ([EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md)):** coverage cabang menemukan pengecekan
> `token.code.length` di `_safeCall` tidak pernah tercapai dari `depositToken` (balanceOf revert duluan), dan
> `test_TokenWithNoCodeIsRejected` lolos karena revert yang salah. Pengecekan dipindah ke awal
> `depositToken`, test diperketat, 5 test cabang + invariant ghost ditambah. Sekarang 17 test, cabang 100%.

## Spesifikasi

- Siapa pun deposit/withdraw ETH (`address(0)`) dan token ERC-20, hanya miliknya sendiri.
- **Invariant:** untuk setiap aset, saldo nyata vault ≥ `totalDeposits[aset]`, dan jumlah saldo
  semua pengguna = `totalDeposits[aset]`.

## Desain `Vault`

| Keputusan | Materi asal |
| --- | --- |
| `mapping(user => mapping(asset => uint256))` + `totalDeposits` | [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md) |
| **Checks-effects-interactions**: saldo dikurangi sebelum transfer keluar | baru (reentrancy) |
| **Kredit selisih saldo**, bukan `amount` yang diminta | baru (fee-on-transfer) |
| `_safeCall`: terima `true` **atau** return kosong, tolak `false`, revert, dan alamat tanpa code | [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) (USDT) |
| `receive()` revert `UseDepositETH` — ETH polos tidak tercatat untuk siapa pun | [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md) |
| Custom error + event `Deposited`/`Withdrawn` | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md), [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) |

## Tiga jebakan: `NaiveVault` vs `Vault`

### 1. Reentrancy

`NaiveVault.withdrawAllETH` mengirim ETH **dulu**, baru menulis saldo = 0. Kontrak penyerang
masuk lagi dari `receive()` selama vault masih punya ETH.

| | NaiveVault | Vault |
| --- | ---: | ---: |
| ETH vault setelah serangan (awal 10 milik Alice + 1 penyerang) | **0** | 10 |
| ETH penyerang | **11** | 1 (miliknya sendiri) |
| Re-entry | 10 kali | gagal (`InsufficientBalance`) |
| Alice masih tercatat punya | 10 ETH (vault kosong) | 10 ETH (aman) |

Catatan: di Solidity 0.8, pola `balance -= amount` setelah call akan revert saat unwind
(underflow) — kebetulan terlindungi. Pola `balance = 0` setelah call (gaya DAO hack) **tidak**
terlindungi. Jangan mengandalkan kebetulan: update state sebelum call.

### 2. Token fee-on-transfer (1%)

Alice dan Bob masing-masing deposit 100.

| | NaiveVault | Vault |
| --- | ---: | ---: |
| Total kredit | 200 | 198 (99 + 99) |
| Saldo nyata vault | 198 | 198 |
| Alice withdraw, lalu Bob withdraw | Bob **revert** — tinggal 98 untuk klaim 100 | keduanya berhasil |

### 3. Token tanpa `bool` (ala USDT)

`NaiveVault`: `require(IERC20Bool(token).transferFrom(...))` → **revert** (return kosong tidak bisa
di-decode jadi `bool`). `Vault`: deposit dan withdraw berhasil lewat `_safeCall`.

## Invariant test

Handler `VaultHandler` punya `deposit` dan `withdraw` dengan argumen acak untuk 3 aktor × 4 aset
(ETH, token normal, token fee, token ala USDT). Foundry memanggilnya 256 run × 500 panggilan.

### Bug di test sendiri: 20% panggilan revert diam-diam

Run pertama lolos, tapi **~25.000 dari 128.000 panggilan revert**. Penyebab: handler memanggil
`MockToken(asset).approve(...)` untuk semua token, termasuk `NoReturnToken` → jebakan 3 menimpa
**kode test**. Akibatnya deposit token ala USDT tidak pernah terjadi — invariant tidak pernah menguji
skenario itu, tapi tetap "lolos".

Perbaikan: approve lewat low-level call. Hasil: **0 revert dari 256.000 panggilan**.

Pencegahan permanen di `foundry.toml`:

```toml
[invariant]
fail_on_revert = true
```

### Apakah invariant bisa menangkap bug? (mutation test)

Bug fee-on-transfer ditanam kembali (kredit `amount`) di salinan project:

| Invariant | Hasil |
| --- | --- |
| `invariant_VaultIsSolvent` | ❌ **"vault holds less than it owes"** (3,98e12 < 4,02e12) |
| `invariant_UserBalancesAddUpToTotal` | ❌ `TokenCallFailed(feeToken)` — withdraw gagal, tertangkap `fail_on_revert` |

Gagal dalam 3 detik. Test yang tidak pernah gagal belum tentu test yang bagus — buktikan dengan
menanam bug.

## Pelajaran Fase 2

1. **Update state sebelum memanggil kontrak lain.**
2. **Jangan percaya angka yang diminta — ukur yang benar-benar diterima.**
3. **Jangan percaya token patuh standar** — perlakukan return value dengan hati-hati.
4. **Revert di handler invariant = area yang tidak diuji.** Aktifkan `fail_on_revert`.
5. **Mutation test** untuk membuktikan test punya gigi.
