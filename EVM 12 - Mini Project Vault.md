---
title: EVM 12 - Mini Project Vault
tags: [learning, evm, solidity, foundry, security, invariant]
updated: 2026-10-24
---

# Mini Project Vault

Sesi 12 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md) — **penutup Fase 2**. Di sesi ini kamu membangun vault
deposit/withdraw ETH + ERC-20 yang menggabungkan Sesi 7–11, membandingkannya dengan versi yang
sengaja salah, lalu mengujinya dengan test unit dan invariant test. Lanjutan dari
[EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md).

Hasil percobaan untuk materi ini ada di [EVM 12 - Hasil Praktik](Hasil%20Praktik/EVM%2012%20-%20Hasil%20Praktik.md).

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

**Jebakan:** pengecekan "alamat tanpa code" harus diletakkan di awal `depositToken`, sebelum
memanggil token apa pun. Kalau diletakkan di dalam `_safeCall`, panggilan `balanceOf` ke alamat tanpa
code revert duluan, jadi pengecekan itu tidak pernah tercapai — dan test yang mengharapkan revert
tetap lolos karena revert yang salah. Coverage cabang membantu menemukan kasus seperti ini (lihat
[EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md)).

## Tiga jebakan: `NaiveVault` vs `Vault`

`NaiveVault` adalah versi yang sengaja salah, untuk menunjukkan apa yang dicegah oleh desain di atas.

### 1. Reentrancy

`NaiveVault.withdrawAllETH` mengirim ETH **dulu**, baru menulis saldo = 0. Kontrak penyerang
masuk lagi dari `receive()` selama vault masih punya ETH, dan setiap re-entry menarik saldo yang
belum dinolkan. Penyerang bisa menguras seluruh ETH vault, termasuk milik pengguna lain, sementara
pengguna lain masih tercatat punya saldo di vault yang sudah kosong.

Di `Vault`, saldo dikurangi sebelum ETH dikirim, jadi re-entry gagal dengan `InsufficientBalance`.

Catatan: di Solidity 0.8, pola `balance -= amount` setelah call akan revert saat unwind
(underflow) — kebetulan terlindungi. Pola `balance = 0` setelah call (gaya DAO hack) **tidak**
terlindungi. Jangan mengandalkan kebetulan: update state sebelum call.

### 2. Token fee-on-transfer

Token yang memotong fee saat transfer membuat vault menerima lebih sedikit dari `amount` yang
diminta. Contoh dengan fee 1%: Alice dan Bob masing-masing deposit 100.

| | NaiveVault | Vault |
| --- | ---: | ---: |
| Total kredit | 200 | 198 (99 + 99) |
| Saldo nyata vault | 198 | 198 |
| Alice withdraw, lalu Bob withdraw | Bob **revert** — tinggal 98 untuk klaim 100 | keduanya berhasil |

### 3. Token tanpa `bool` (ala USDT)

`NaiveVault`: `require(IERC20Bool(token).transferFrom(...))` → **revert** (return kosong tidak bisa
di-decode jadi `bool`). `Vault`: deposit dan withdraw berhasil lewat `_safeCall`.

## Invariant test

Handler (`VaultHandler`) punya `deposit` dan `withdraw` dengan argumen acak untuk beberapa aktor dan
beberapa aset (ETH, token normal, token fee, token ala USDT). Foundry memanggilnya berkali-kali
(`runs` × `depth`) dengan urutan acak, lalu mengecek invariant setelah tiap panggilan.

### Jebakan: revert di handler = area yang tidak diuji

Secara default, panggilan handler yang revert diabaikan diam-diam. Kalau handler punya bug —
misalnya memanggil `approve` dengan interface ber-`bool` ke token ala USDT, sehingga jebakan 3
menimpa **kode test** — sebagian besar panggilan untuk aset itu revert. Skenario itu tidak pernah
benar-benar diuji, tapi invariant tetap "lolos".

Perbaikan di handler: approve token ala USDT lewat low-level call. Pencegahan permanen di
`foundry.toml`:

```toml
[invariant]
fail_on_revert = true
```

### Apakah invariant bisa menangkap bug? (mutation test)

Test yang tidak pernah gagal belum tentu test yang bagus. Buktikan dengan **menanam bug** di salinan
project — misalnya kembalikan bug fee-on-transfer (kredit `amount`, bukan selisih saldo) — lalu
jalankan invariant. Test yang bagus harus gagal dengan cepat:

- `invariant_VaultIsSolvent` gagal karena vault memegang lebih sedikit dari yang ia utang.
- `invariant_UserBalancesAddUpToTotal` gagal karena withdraw token fee revert, dan tertangkap
  `fail_on_revert`.

## Pelajaran Fase 2

1. **Update state sebelum memanggil kontrak lain.**
2. **Jangan percaya angka yang diminta — ukur yang benar-benar diterima.**
3. **Jangan percaya token patuh standar** — perlakukan return value dengan hati-hati.
4. **Revert di handler invariant = area yang tidak diuji.** Aktifkan `fail_on_revert`.
5. **Mutation test** untuk membuktikan test punya gigi.

## Coba sendiri

1. Tulis `Vault` sesuai desain di atas, `NaiveVault` yang sengaja salah, dan mock token: token normal,
   token fee 1%, dan token tanpa return value.
2. Tulis test unit untuk ketiga jebakan: serangan reentrancy, dua penarik token fee, dan token ala
   USDT.
3. Tulis handler invariant, aktifkan `fail_on_revert`, lalu jalankan invariant test.
4. Tanam kembali bug fee-on-transfer di salinan project, dan pastikan invariant gagal.

```bash
forge test -vv                                    # test unit + invariant
forge test --match-contract VaultInvariant -vv    # invariant saja
forge coverage --report summary                   # cek cabang yang belum tercapai
```
