---
title: EVM 19 - Testing Lanjutan
tags: [learning, evm, foundry, testing, fuzz, invariant, coverage]
updated: 2026-09-24
---

# Testing Lanjutan

Catatan dari Sesi 19 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24 — sesi pertama Fase 4. Dasar fuzz/invariant/fork
sudah dipakai di [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md), [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md), [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md).
Sesi ini: coverage, differential testing, ghost variable, shrinking, fork test yang rapi. Lanjutan dari
[EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md).

Kode: `~/Documents/riset/evm/vault-project` (coverage, ghost) dan `~/Documents/riset/evm/swap`
(`src/AmmMath.sol`, `test/Differential.t.sol`).

## 1. Coverage: baris 100% bisa menipu

```bash
forge coverage --no-match-contract VaultInvariantTest --report summary
forge coverage --no-match-contract VaultInvariantTest --report lcov    # detail per cabang (BRDA)
```

`Vault.sol` sebelum: **100% baris, 40% cabang (4/10)**. Setiap baris pernah jalan, tapi enam `if … revert`
hanya pernah diuji satu arah.

### Temuan: pengecekan mati + test yang lolos karena alasan salah

`_safeCall` punya `if (token.code.length == 0) revert TokenCallFailed(token)`. Tapi `depositToken` memanggil
`balanceOf(token)` **sebelum** `_safeCall` — pada alamat tanpa kode, `balanceOf` sudah revert saat decode.
Pengecekan tidak pernah tercapai.

Test Sesi 12 `test_TokenWithNoCodeIsRejected` memakai `vm.expectRevert()` **tanpa argumen** → lolos oleh
revert yang salah. **`expectRevert()` kosong itu lemah**: tidak membedakan revert yang dirancang dari
revert kebetulan.

Perbaikan: cek kode di awal `depositToken`; test lama diperketat ke `TokenCallFailed(notAToken)`; lima test
baru (deposit 0, withdraw 0, ETH ke kontrak yang menolak, token yang revert, token yang return `false` —
mock `FalseReturnToken`).

Sesudah: **`Vault.sol` 100% baris, 100% cabang (10/10)**, 17 test lolos, invariant tetap lolos.

## 2. Differential testing

Implementasi referensi = router Uniswap v2 asli di fork (`getAmountOut` adalah fungsi `pure`).

| Versi | Rumus | Test manual (1 ETH, pool Sesi 13) | Fuzz 256 run vs router |
| --- | --- | --- | --- |
| `getAmountOut` | `in×997×R_out / (R_in×1000 + in×997)` | sama | **lolos** |
| `getAmountOutEarlyFee` | `f = in×997/1000; f×R_out / (R_in + f)` | **sama (2.656.276.900)** | **gagal** |

Di atas kertas identik, tapi versi kedua membulatkan dua kali:

| Kasus | Router | Fee duluan |
| --- | ---: | ---: |
| input 1 wei, reserve 1 / 1e18 | 499.248.873.309.964.947 | **0** |
| input 999, reserve 1e6 / 1e24 | …768.845 | selisih 2,99 × 10¹⁵ |
| input 1.234.567, reserve 1e12 / 1e30 | …219.547 | selisih 2,99 × 10¹⁷ |
| 1 ETH, pool Sesi 13 | 2.656.276.900 | **sama** |
| angka raksasa dari trace fuzz | …036.222 | selisih 1 |

**Test manual menguji kasus yang terpikir; fuzz menguji yang tidak terpikir.** Differential test tidak
butuh jawaban benar per kasus — cukup implementasi referensi.

## 3. Ghost variable dan shrinking

Handler invariant mencatat sendiri token yang benar-benar masuk/keluar vault (`ghostIn`, `ghostOut`,
diukur dari selisih saldo). Invariant baru: `totalDeposits == ghostIn − ghostOut` — lebih tajam dari
"vault solvent".

Mutation: `withdraw` lupa mengurangi `totalDeposits` saat saldo ditarik penuh.

| | |
| --- | --- |
| Hasil | `[FAIL: books != real flows: 1 != 0]` |
| Urutan panggilan | **original 17 → shrunk 2** (`deposit`, lalu `withdraw` seluruh saldo) |
| Replay | urutan gagal disimpan di `cache/invariant/failures/`, diulang duluan di run berikutnya |

Shrinking mengubah 17 langkah acak jadi 2 langkah yang langsung menunjuk bug.

`afterInvariant()` dipanggil setelah tiap run — tempat mencetak statistik handler (jumlah deposit/withdraw).

## 4. Fork test yang rapi

```toml
# foundry.toml
[rpc_endpoints]
mainnet = "https://eth.drpc.org"
```

```solidity
function setUp() public {
    vm.createSelectFork("mainnet", 26_046_500); // blok di-pin: hasil sama di setiap run
}
```

| | Waktu |
| --- | ---: |
| Run 1 (cache kosong) | 1,56 s |
| Run 2 | **0,27 s** |

Cache per blok di `~/.foundry/cache/rpc/mainnet/<blok>/`. Blok yang di-pin = test deterministik + cepat.
Tanpa pin, setiap run memakai blok terbaru dan hasilnya bisa berubah.

## Ringkasan teknik

| Teknik | Menjawab | Kapan dipakai |
| --- | --- | --- |
| Unit test | kasus yang terpikir | selalu |
| Fuzz | input acak untuk satu fungsi | fungsi matematis, batas nilai |
| Invariant + handler | urutan panggilan acak, multi-aktor | kontrak yang punya state |
| Ghost variable | "apakah pembukuan = aliran nyata?" | vault, lending, AMM |
| Differential | "apakah sama dengan referensi?" | port rumus, optimasi gas |
| Coverage cabang | "cabang mana belum pernah diuji?" | sebelum audit |
| Mutation | "apakah test bisa gagal?" | membuktikan test punya gigi |
