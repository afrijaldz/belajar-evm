---
title: EVM 05 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 05 - Hasil Praktik

Hasil percobaan untuk materi [EVM 05 - Fork dan Foundry](../EVM%2005%20-%20Fork%20dan%20Foundry.md). Dijalankan 2026-09-24: `anvil` fork
BSC di blok **123.136.365** (satu blok sebelum transaksi burn mingguan CAKE 2026-09-21), `cast`,
dan `forge test` (Foundry 1.7.1). Project: `~/Documents/riset/evm/account-model`, test
`test/CakeBurnFork.t.sol`.

```bash
anvil --fork-url <rpc-arsip> --fork-block-number 123136365
```

## Uji RPC BSC gratis untuk fork (2026-09-24)

| RPC | Hasil |
| --- | --- |
| `bsc-mainnet.nodereal.io/v1/${NODEREAL_BSC_API_KEY}` | ✅ **archive, berhasil** — tapi rate limit (HTTP 429). Wajib `--compute-units-per-second 20` |
| `bsc.meowrpc.com` | ❌ bukan full archive: `state at block … is pruned` setelah ~3 hari; akun validator gagal diambil → **semua transaksi macet**, termasuk transfer polos |
| `bsc-mainnet.public.blastapi.io` | ❌ `Only core evm requests are allowed` untuk metode akun `anvil` |
| `1rpc.io/bnb` | ❌ `eth_getBalance` historis tidak didukung |
| `bsc-dataseed.bnbchain.org` | ❌ `missing trie node` (full node biasa) |
| `bsc-rpc.publicnode.com` | ❌ archive butuh token pribadi |

## Impersonate Timelock

Owner MasterChefV2 = Timelock `0xA1f4…fE4`. Timelock di-impersonate, diberi 10 BNB, lalu
memanggil `burnCake(false)` ke MasterChefV2.

**Hasil `burnCake(false)` vs transaksi asli 2026-09-21:**

| | Fork | Asli |
| --- | ---: | ---: |
| Mint ke Safe burn | 530.328 | 530.328 ✓ |
| Mint ke SyrupBar → MCv2 | 5.303.280 | 5.303.280 ✓ |
| MCv2 → Safe burn | 54.049.639,92 | 54.049.639,92 ✓ |
| Sisa MCv2 | 388.910 | 388.910 ✓ |
| Saldo Safe burn sesudahnya | **60.089.238** | = burn 7 hari ke `0xdead` di CAKE Diperiksa Ulang 2026-09-24 ✓ |
| gasUsed | 135.915 | 2.454.872 (`true`) |

Selisih ~2,3 jt gas = `massUpdatePools` (versi `true`). Versi itu menyentuh ribuan slot dan
membuat fork macet di RPC publik. `false` selesai dalam **4 detik**.

## Menulis storage CAKE

Mencari slot mapping `balances` CAKE:

| Index | Hasil |
| --- | --- |
| 0 | 0 (slot 0 = `_owner` dari `Ownable`) |
| **1** | **cocok** → `balances` di slot 1 |

```bash
cast rpc anvil_setStorageAt <cake> $(cast index address <saya> 1) $(cast to-uint256 1000000000000000000000)
```

Hasil: saldo saya 0 → **1.000 CAKE**, transfer 100 ke `0xdead` sukses, tapi `totalSupply` tidak
berubah.

## Test Foundry di atas fork

```bash
forge test --match-contract CakeBurnForkTest \
  --fork-url <nodereal> --fork-block-number 123136365 --compute-units-per-second 20 -vv
```

| Test | Hasil |
| --- | --- |
| `test_EffectiveSupplyIsSmallFractionOfTotalSupply` | PASS — total 5,382 M, efektif 385,3 jt |
| `test_OnlyTimelockCanBurn` | PASS — `vm.expectRevert("Ownable: caller is not the owner")` |
| `test_BurnCakeMovesWeeklyStockToBurnSafe` | PASS — 54.579.924 pindah (54.049.640 + 530.328 mint langsung) |

3 test, **5 detik**. `vm.prank(TIMELOCK)` dipakai untuk impersonate.

## Temuan untuk riset CAKE

### Pertanyaan terbuka Sesi 4 terjawab

MCv2 menerima cuma 5,3 jt di transaksi burn tapi mengirim 54 jt (lihat
[EVM 04 - Hasil Praktik](EVM%2004%20-%20Hasil%20Praktik.md)) — karena **menumpuk sepanjang minggu**:

| Waktu (UTC) | Saldo CAKE MCv2 |
| --- | ---: |
| Sen 14 Sep 06:53 | 484.210 |
| Kam 17 Sep 06:53 | 362.332 |
| Jum 18 Sep 06:52 | **26.753.257** |
| Min 20 Sep 06:52 | **43.215.035** |
| Sen 21 Sep 06:51 (sebelum burn) | **49.135.270** |
| 2 blok sesudah burn | 388.910 |

Lompatan = panen dari MasterChefV1 oleh transaksi lain. Tiap Senin `burnCake` mengosongkannya.

### Aliasing sampel

Sampel bulanan di CAKE Diperiksa Ulang 2026-09-24 kebetulan jatuh di "lembah" siklus
mingguan (MCv2 ~0,37 jt di 6 titik yang dicek).

**Rumus supply efektif yang benar** juga harus mengurangi saldo MCv2:
`totalSupply − 0xdead − Safe burn − MasterChefV2`. Di blok fork ini tanpa MCv2 = 385,3 jt,
dengan MCv2 = 336,2 jt. Deret −8,25% tetap valid karena titik sampelnya kebetulan bersih, tapi
sampel di hari Minggu bisa melenceng sampai +13%.

## Jebakan yang ditemui

Semua jebakan di materi (checksum EIP-55, `emit` di `view`, timeout `cast send`, `pkill -f`)
ditemui di sesi ini. Satu lagi yang khusus lingkungan kerja: harness agent menolak `sleep`
panjang di foreground → pakai proses background + `until` loop.
