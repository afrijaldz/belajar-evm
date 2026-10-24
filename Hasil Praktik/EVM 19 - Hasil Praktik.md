---
title: EVM 19 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 19 - Hasil Praktik

Hasil percobaan untuk materi [EVM 19 - Testing Lanjutan](../EVM%2019%20-%20Testing%20Lanjutan.md). Dijalankan 2026-09-24. Kode:
`~/Documents/riset/evm/vault-project` (coverage, ghost) dan `~/Documents/riset/evm/swap`
(`src/AmmMath.sol`, `test/Differential.t.sol`).

## Coverage `Vault.sol`

Sebelum: **100% baris, 40% cabang (4/10)**. Enam `if … revert` hanya pernah diuji satu arah.

### Temuan: pengecekan mati + test yang lolos karena alasan salah

`_safeCall` punya `if (token.code.length == 0) revert TokenCallFailed(token)`. Tapi `depositToken` memanggil
`balanceOf(token)` **sebelum** `_safeCall` — pada alamat tanpa kode, `balanceOf` sudah revert saat decode.
Pengecekan tidak pernah tercapai.

Test Sesi 12 `test_TokenWithNoCodeIsRejected` memakai `vm.expectRevert()` **tanpa argumen** → lolos oleh
revert yang salah.

Perbaikan: cek kode di awal `depositToken`; test lama diperketat ke `TokenCallFailed(notAToken)`; lima test
baru (deposit 0, withdraw 0, ETH ke kontrak yang menolak, token yang revert, token yang return `false` —
mock `FalseReturnToken`).

Sesudah: **`Vault.sol` 100% baris, 100% cabang (10/10)**, 17 test lolos, invariant tetap lolos.

## Differential testing

| Versi | Test manual (1 ETH, pool Sesi 13) | Fuzz 256 run vs router |
| --- | --- | --- |
| `getAmountOut` | sama | **lolos** |
| `getAmountOutEarlyFee` | **sama (2.656.276.900)** | **gagal** |

| Kasus | Router | Fee duluan |
| --- | ---: | ---: |
| input 1 wei, reserve 1 / 1e18 | 499.248.873.309.964.947 | **0** |
| input 999, reserve 1e6 / 1e24 | …768.845 | selisih 2,99 × 10¹⁵ |
| input 1.234.567, reserve 1e12 / 1e30 | …219.547 | selisih 2,99 × 10¹⁷ |
| 1 ETH, pool Sesi 13 | 2.656.276.900 | **sama** |
| angka raksasa dari trace fuzz | …036.222 | selisih 1 |

## Ghost variable dan shrinking

Mutation: `withdraw` lupa mengurangi `totalDeposits` saat saldo ditarik penuh.

| | |
| --- | --- |
| Hasil | `[FAIL: books != real flows: 1 != 0]` |
| Urutan panggilan | **original 17 → shrunk 2** (`deposit`, lalu `withdraw` seluruh saldo) |
| Replay | urutan gagal disimpan di `cache/invariant/failures/`, diulang duluan di run berikutnya |

## Fork test di-pin (blok 26.046.500)

| | Waktu |
| --- | ---: |
| Run 1 (cache kosong) | 1,56 s |
| Run 2 | **0,27 s** |
