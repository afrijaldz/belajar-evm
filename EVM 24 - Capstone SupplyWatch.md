---
title: EVM 24 - Capstone SupplyWatch
tags: [learning, evm, capstone, solidity, tokenomics, create2]
updated: 2026-09-24
---

# Capstone: SupplyWatch

Catatan dari Sesi 24 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24 — **penutup modul**. Satu kontrak yang menggabungkan
Sesi 1–23: mencatat supply efektif token dan laju burn-nya **on-chain**, tanpa owner, bisa diaudit siapa
pun. Lanjutan dari [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md).

Project: `~/Documents/riset/evm/capstone` — `src/SupplyWatch.sol`, `test/SupplyWatch.t.sol` (6 unit + 2 fork),
`script/DeploySupplyWatch.s.sol`.

```bash
cd ~/Documents/riset/evm/capstone
forge test --match-contract SupplyWatchUnitTest -vv
forge test --match-test test_UniLast30Days -vv
forge test --match-test test_CakeLast12Months --compute-units-per-second 20 -vv
```

## Masalah yang dijawab

Riset tokenomics memakai angka supply/burn dari agregator (sering salah — CoinGecko di UNI, HYPE, ETHFI) atau
dari script lokal (tidak bisa diverifikasi orang lain). `SupplyWatch` memindahkan perhitungannya on-chain.

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

### Unit (6, semua lolos)

| Test | Membuktikan |
| --- | --- |
| `BothBurnStylesGiveTheSameEffectiveSupply` | `burn()` 100k + kirim 100k ke `0xdead` → `totalSupply` 900k, efektif **800k** |
| `CheckpointAtMostOncePerDay` | kedua kali di hari sama → `TooSoon`; setelah 1 hari → boleh |
| `AnnualizedChange` | −2% dalam 73 hari → **−1.000 bps/tahun** |
| `CheckpointFitsOneSlot` | dibaca langsung dari storage: supply di 128 bit atas, slot berikutnya kosong |
| `UnknownViewAndTooManyExcluded` | error yang benar |
| `testFuzz_EffectiveNeverAboveTotal` (256 run) | efektif ≤ `totalSupply` dan = rumus |

### Fork dengan token asli

**UNI** (Ethereum, blok 25.830.000 → 26.046.700, 30 hari, `vm.rollFork` + `vm.makePersistent`):

| View | Awal → akhir | Laju |
| --- | --- | ---: |
| `totalSupply − 0xdead` | 890 jt → 887 jt | **−3,73%/th** |
| `− 0xdead − treasury timelock` | 623 jt → 620 jt | **−5,33%/th** |

Lebih cepat dari −1,53% setahun di UNI Diperiksa dengan Saringan Lima Chain — revenue melonjak sejak
fee switch v4 akhir Juli.

**CAKE** (BSC, blok 62.230.887 → 123.707.633, 12 bulan), view `− 0xdead − Safe burn − MasterChefV2`:

| Awal → akhir | Laju |
| --- | ---: |
| 366 jt → 336 jt | **−825 bps = −8,25%/th** |

**Persis sama dengan CAKE Diperiksa Ulang 2026-09-24** — riset terverifikasi ulang dengan metode yang
sama sekali berbeda.

## Deploy CREATE2

`new SupplyWatch{salt: keccak256("afrijal.evm-learning.supply-watch.v1")}()` di forge script.

| Chain | Alamat (simulasi, tidak di-broadcast) |
| --- | --- |
| prediksi `cast create2` | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Ethereum (1) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Base (8453) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Arbitrum (42161) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |

**Konsekuensi keamanan:** lewat deployer Arachnid, alamat tidak bergantung pada wallet — **siapa pun** bisa
men-deploy bytecode ini ke alamat itu di chain mana pun. Aman untuk `SupplyWatch` (tanpa owner, tanpa argumen
constructor: hasilnya selalu kontrak yang sama). **Berbahaya** untuk kontrak yang menetapkan owner dari
constructor — penyerang bisa men-deploy duluan dengan owner-nya sendiri.

## Ukuran dan gas

| | |
| --- | ---: |
| Runtime | 3.669 byte |
| Deploy | ~846.648 gas |
| `register` | ~139.000 gas |
| `checkpoint` | ~70.000–85.000 gas |
| `effectiveSupply` (view) | ~17.600 gas |

## Jebakan yang ditemui

- **Checksum alamat Safe burn salah lagi** — disalin dari versi Sesi 5 yang dulu salah ketik. Semua alamat
  kini divalidasi `cast to-check-sum-address`.
- **drpc gratis** bisa `eth_call` historis tapi gagal/timeout saat fork membaca akun & storage lama (HTTP 408,
  500). **Tenderly** (`mainnet.gateway.tenderly.co`) lolos keduanya.
- `sed` dengan pemisah `#` gagal kalau teks penggantinya berisi `#`.
- Alamat test contract Foundry `0x5615…b72f` punya saldo di mainnet (ada yang pernah mengirim ke sana).

## Belum dikerjakan

Deploy sungguhan butuh saldo: dijalankan bersama Sesi 21 setelah faucet masuk
([EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md)).
