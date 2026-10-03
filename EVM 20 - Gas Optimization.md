---
title: EVM 20 - Gas Optimization
tags: [learning, evm, solidity, gas, foundry]
updated: 2026-09-24
---

# Gas Optimization

Catatan dari Sesi 20 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Target: `Vault.sol` dari
[EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) — jaring pengaman terlengkap (17 test, 3 invariant, cabang 100% dari
[EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md)). Satu perubahan per langkah; setiap langkah wajib lolos semua test.

Project: `~/Documents/riset/evm/vault-project` — `bench.py` (benchmark transaksi nyata di anvil port 8546),
`.gas-snapshot` (baseline `forge snapshot`).

```bash
cd ~/Documents/riset/evm/vault-project
python3 bench.py                                              # gas transaksi nyata, skenario tetap
forge snapshot --no-match-contract VaultInvariantTest --diff  # dibanding snapshot tersimpan
forge test --gas-report                                       # per fungsi (dalam test: cold/warm campur)
```

## Metode

1. Baseline dulu: `forge snapshot` + gas report + `bench.py`.
2. Satu perubahan → semua test + invariant → `bench.py`.
3. Angka yang dipercaya = **transaksi nyata** (pelajaran [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md), [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md)).

## Hasil (gas transaksi nyata)

| Langkah | Deploy | `depositETH` pertama | `depositETH` lagi | `withdraw` ETH | `depositToken` pertama | `depositToken` lagi | `withdraw` token |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 baseline (optimizer off) | 916.426 | 68.125 | 33.925 | 42.086 | 112.620 | 61.320 | 52.521 |
| 1 **optimizer on** | 511.255 (−44%) | 67.668 | 33.468 | 41.172 | 109.003 | 57.703 | 50.497 |
| 2 `abi.encodeCall` | **sama persis** | | | | | | |
| 3 `unchecked` (2 pengurangan) | 503.913 | 67.668 | 33.468 | 40.998 | 109.003 | 57.703 | 50.323 |
| (4) tanpa `totalDeposits` — **tidak dipakai** | 460.645 | **45.424 (−33%)** | 28.324 | 35.904 | 86.714 | 52.514 | 45.229 |
| (5) `via_ir` — **tidak dipakai** | **385.220 (−58%)** | 67.611 | 33.411 | 40.822 | 108.133 | 56.833 | 49.859 |

`forge snapshot --diff` setelah langkah 1–3: 17/17 test lebih murah, total −23% (termasuk deploy di `setUp`).

## Pelajaran

1. **Optimizer = kemenangan gratis terbesar untuk deploy** (−44%). Foundry mematikannya secara bawaan.
   `optimizer_runs` tinggi → runtime lebih murah, deploy lebih mahal; rendah → sebaliknya.
2. **Runtime didominasi storage dan panggilan eksternal.** Compiler cuma bisa memotong 1–6%.
3. **Tips populer bisa bernilai nol.** `abi.encodeWithSignature("…")` → `abi.encodeCall`: selisih **0 gas**,
   karena optimizer sudah menghitung keccak string literal saat kompilasi. Tetap dipakai karena `encodeCall`
   dicek tipenya oleh compiler.
4. **`unchecked` hanya dengan bukti.** Hemat 174 gas per `withdraw`. Setiap blok `unchecked` diberi komentar
   *kenapa* aman dan invariant mana yang menjaminnya.
5. **Pengungkit terbesar adalah desain: jumlah slot storage.** Menghapus `totalDeposits` = −22.244 gas pada
   deposit pertama (satu slot baru 20.000, lihat [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)). Tapi `totalDeposits`
   memungkinkan siapa pun mengecek solvency on-chain dan dipakai invariant. **Keputusan: dipertahankan** —
   untuk vault yang memegang dana orang lain, auditabilitas sepadan dengan harganya. Alternatifnya
   pembukuan lewat event (murah, [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md)) tapi hanya bisa dicek off-chain.
6. **`via_ir`**: deploy −24% lagi, runtime cuma beberapa ratus gas, kompilasi lebih lambat. Tidak dipakai.

## Yang sudah otomatis optimal dari sesi sebelumnya

- Custom error, bukan `require` string ([EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md)).
- `immutable` untuk alamat yang tidak berubah ([EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md)).
- Mapping bersarang, bukan array yang di-loop.
- Event untuk riwayat, storage hanya untuk keadaan sekarang.
