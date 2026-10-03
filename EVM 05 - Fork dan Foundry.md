---
title: EVM 05 - Fork dan Foundry
tags: [learning, evm, bsc, foundry, fork]
updated: 2026-09-24
---

# Fork dan Foundry EVM

Catatan dari Sesi 5 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Semua hasil observasi langsung: `anvil` fork BSC
di blok **123.136.365** (satu blok sebelum transaksi burn mingguan CAKE 2026-09-21), `cast`, dan
`forge test` (Foundry 1.7.1). Lanjutan dari [EVM 04 - ABI](EVM%2004%20-%20ABI.md).

Project praktik: `~/Documents/riset/evm/account-model`, test `test/CakeBurnFork.t.sol`.

## Apa itu fork

```bash
anvil --fork-url <rpc-arsip> --fork-block-number 123136365
```

- Chain lokal yang **meminjam state chain asli** di blok tertentu. Chain ID tetap 56.
- State diambil **lazy**: hanya slot yang disentuh yang diminta ke RPC, lalu di-cache.
- Semua perubahan hanya lokal. Akun `anvil` otomatis dapat 10.000 BNB palsu.
- **Pin nomor blok** supaya hasil bisa diulang persis.

## Syarat utama: archive node

Fork di blok lama butuh RPC **archive**. Full node hanya menyimpan state ~128 blok terakhir:
di BSC (blok 0,45 detik) itu **kurang dari 1 menit**, di Ethereum (12 detik) ~25 menit.

Hasil uji RPC BSC gratis untuk fork (2026-09-24):

| RPC | Hasil |
| --- | --- |
| `bsc-mainnet.nodereal.io/v1/<NODEREAL_API_KEY>` | ✅ **archive, berhasil** — tapi rate limit (HTTP 429). Wajib `--compute-units-per-second 20` |
| `bsc.meowrpc.com` | ❌ bukan full archive: `state at block … is pruned` setelah ~3 hari; akun validator gagal diambil → **semua transaksi macet**, termasuk transfer polos |
| `bsc-mainnet.public.blastapi.io` | ❌ `Only core evm requests are allowed` untuk metode akun `anvil` |
| `1rpc.io/bnb` | ❌ `eth_getBalance` historis tidak didukung |
| `bsc-dataseed.bnbchain.org` | ❌ `missing trie node` (full node biasa) |
| `bsc-rpc.publicnode.com` | ❌ archive butuh token pribadi |

Gejala RPC tidak cocok: transaksi **pending selamanya** (`txpool_status` → `pending: 0x1`) walau
`cast call` masih jalan. Cek dengan transfer polos dulu sebelum menyalahkan kontraknya.

## Cara 1: menyamar (impersonate)

Owner MasterChefV2 = Timelock `0xA1f4…fE4`. Di fork, siapa pun bisa jadi siapa pun:

```bash
cast rpc anvil_impersonateAccount <timelock>
cast rpc anvil_setBalance <timelock> 0x8AC7230489E80000    # 10 BNB untuk gas
cast send <masterchef> "burnCake(bool)" false --from <timelock> --unlocked
```

Tanpa menyamar: revert `Ownable: caller is not the owner` — datanya diawali `0x08c379a0`,
selector `Error(string)`. Pesan revert juga di-encode ABI (lihat [EVM 04 - ABI](EVM%2004%20-%20ABI.md)).

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
**membuat fork macet** di RPC publik. `false` selesai dalam **4 detik**.

## Cara 2: menulis storage

Mencari slot mapping `balances` CAKE: hitung `keccak256(alamat, i)` untuk holder yang saldonya
diketahui, cocokkan dengan `balanceOf`.

| Index | Hasil |
| --- | --- |
| 0 | 0 (slot 0 = `_owner` dari `Ownable`) |
| **1** | **cocok** → `balances` di slot 1 |

```bash
cast rpc anvil_setStorageAt <cake> $(cast index address <saya> 1) $(cast to-uint256 1000000000000000000000)
```

Hasil: saldo saya 0 → **1.000 CAKE**, transfer 100 ke `0xdead` sukses, tapi **`totalSupply` tidak
berubah** — jumlah saldo kini > `totalSupply`. Menulis storage melewati semua logika dan bisa
merusak invariant. Cheatcode `deal()` Foundry melakukan hal yang sama.

`evm_snapshot` / `evm_revert` mengembalikan state seketika — pakai untuk mengulang skenario.

## Cara 3: test Foundry di atas fork

```bash
forge test --match-contract CakeBurnForkTest \
  --fork-url <nodereal> --fork-block-number 123136365 --compute-units-per-second 20 -vv
```

| Test | Hasil |
| --- | --- |
| `test_EffectiveSupplyIsSmallFractionOfTotalSupply` | PASS — total 5,382 M, efektif 385,3 jt |
| `test_OnlyTimelockCanBurn` | PASS — `vm.expectRevert("Ownable: caller is not the owner")` |
| `test_BurnCakeMovesWeeklyStockToBurnSafe` | PASS — 54.579.924 pindah (54.049.640 + 530.328 mint langsung) |

3 test, **5 detik**. `vm.prank(TIMELOCK)` = impersonate untuk satu panggilan.

## Temuan untuk riset CAKE

### Pertanyaan terbuka Sesi 4 terjawab

MCv2 menerima cuma 5,3 jt di transaksi burn tapi mengirim 54 jt — karena **menumpuk sepanjang
minggu**:

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
mingguan (MCv2 ~0,37 jt di 6 titik yang dicek). **Sampling berkala bisa tertipu siklus yang
lebih pendek dari intervalnya.**

**Rumus supply efektif yang benar** juga harus mengurangi saldo MCv2:
`totalSupply − 0xdead − Safe burn − MasterChefV2`. Di blok fork ini tanpa MCv2 = 385,3 jt,
dengan MCv2 = 336,2 jt. Deret −8,25% tetap valid karena titik sampelnya kebetulan bersih, tapi
sampel di hari Minggu bisa melenceng sampai +13%.

## Jebakan yang ditemui

- **Checksum alamat (EIP-55).** Solidity menolak alamat dengan huruf besar-kecil salah. Verifikasi
  dengan `cast to-check-sum-address`. Hati-hati regex 40 hex juga menangkap potongan hash transaksi.
- **`emit` tidak boleh di fungsi `view`** — log adalah perubahan state.
- `cast send` menunggu konfirmasi dengan timeout. Untuk fork lambat: `--async`, lalu poll
  `cast receipt`.
- Harness menolak `sleep` panjang di foreground → pakai proses background + `until` loop.
- `pkill -f <pola>` bisa membunuh shell sendiri; pakai trik `[b]urn_retry.sh` atau `pkill -x anvil`.
