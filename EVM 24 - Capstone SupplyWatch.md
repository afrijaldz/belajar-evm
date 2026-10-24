---
title: EVM 24 - Capstone SupplyWatch
tags: [learning, evm, capstone, solidity, tokenomics, create2]
updated: 2026-10-24
---

# Capstone: SupplyWatch

Sesi 24 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md) — **penutup modul**. Kamu membangun satu kontrak yang menggabungkan Sesi
1–23: mencatat supply efektif token dan laju burn-nya **on-chain**, tanpa owner, bisa diaudit siapa pun.
Lanjutan dari [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md).

Hasil percobaan untuk materi ini ada di [EVM 24 - Hasil Praktik](Hasil%20Praktik/EVM%2024%20-%20Hasil%20Praktik.md).

## Masalah yang dijawab

Riset tokenomics memakai angka supply/burn dari agregator (sering salah) atau dari script lokal (tidak bisa
diverifikasi orang lain). `SupplyWatch` memindahkan perhitungannya on-chain.

## Desain

| Keputusan | Dari sesi |
| --- | --- |
| Supply efektif = `totalSupply − Σ balanceOf(dikecualikan)` — benar untuk kedua gaya burn | [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md) |
| **Tanpa owner.** Siapa pun `register` view (token + daftar pengecualian + label); **tidak bisa diubah** setelahnya. Tidak setuju? daftarkan view sendiri | [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md), [EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md) |
| `checkpoint` bisa dipanggil siapa pun, **maks 1× per hari** per view | [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md) (aliasing), [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md) |
| Laju dihitung dari **`block.timestamp`** | [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md) (`block.number` Arbitrum = L1) |
| Checkpoint **1 slot**: `uint64 timestamp`, `uint64 blockNumber`, `uint128 supply` | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md), [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md) |
| Maks 16 alamat dikecualikan per view (batas gas loop) | [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md) |
| Custom error, event untuk setiap register/checkpoint | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md), [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) |
| Deploy **CREATE2** lewat deployer Arachnid → alamat sama di semua chain | [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md) |

## Test

### Unit

| Test | Yang harus dibuktikan |
| --- | --- |
| `BothBurnStylesGiveTheSameEffectiveSupply` | `burn()` 100k + kirim 100k ke `0xdead` dari supply 1 jt → `totalSupply` 900k, efektif **800k** |
| `CheckpointAtMostOncePerDay` | kedua kali di hari sama → `TooSoon`; setelah 1 hari → boleh |
| `AnnualizedChange` | −2% dalam 73 hari → **−1.000 bps/tahun** |
| `CheckpointFitsOneSlot` | dibaca langsung dari storage: supply di 128 bit atas, slot berikutnya kosong |
| `UnknownViewAndTooManyExcluded` | error yang benar |
| `testFuzz_EffectiveNeverAboveTotal` (256 run) | efektif ≤ `totalSupply` dan = rumus |

### Fork dengan token asli

Untuk mengukur laju di masa lalu, test fork berpindah blok dengan `vm.rollFork`. Kontrak `SupplyWatch` yang
di-deploy di blok awal harus ditandai `vm.makePersistent` supaya tetap ada setelah fork berpindah blok.

- **UNI** (Ethereum, 30 hari): bandingkan view `totalSupply − 0xdead` dengan view yang juga mengecualikan
  treasury timelock.
- **CAKE** (BSC, 12 bulan): view `− 0xdead − Safe burn − MasterChefV2`.

Hasil fork bisa dibandingkan dengan angka riset tokenomics yang dihitung dengan cara lain. Kalau dua metode yang
sama sekali berbeda memberi angka yang sama, angkanya terverifikasi ulang.

## Deploy CREATE2

`new SupplyWatch{salt: keccak256("afrijal.evm-learning.supply-watch.v1")}()` di forge script. Alamatnya bisa
diprediksi dengan `cast create2`, dan simulasi di chain mana pun harus memberi alamat yang sama.

**Konsekuensi keamanan:** lewat deployer Arachnid, alamat tidak bergantung pada wallet — **siapa pun** bisa
men-deploy bytecode ini ke alamat itu di chain mana pun. Aman untuk `SupplyWatch` (tanpa owner, tanpa argumen
constructor: hasilnya selalu kontrak yang sama). **Berbahaya** untuk kontrak yang menetapkan owner dari
constructor — penyerang bisa men-deploy duluan dengan owner-nya sendiri.

## Jebakan

- **Checksum alamat yang disalin dari catatan lama** bisa salah ketik. Validasi semua alamat dengan
  `cast to-check-sum-address`.
- **RPC gratis** (mis. drpc) bisa melayani `eth_call` historis tapi gagal/timeout saat fork membaca akun &
  storage lama (HTTP 408, 500). Tenderly (`mainnet.gateway.tenderly.co`) melayani keduanya.
- `sed` dengan pemisah `#` gagal kalau teks penggantinya berisi `#`.
- Alamat test contract bawaan Foundry `0x5615…b72f` punya saldo di mainnet (ada yang pernah mengirim ke
  sana). Di fork test, jangan anggap saldo kontrak test = 0.

## Coba sendiri

```bash
forge test --match-contract SupplyWatchUnitTest -vv
forge test --match-test test_UniLast30Days -vv
forge test --match-test test_CakeLast12Months --compute-units-per-second 20 -vv
```

## Tugas

Deploy sungguhan butuh saldo: jalankan bersama Sesi 21 setelah faucet masuk
([EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md)).
