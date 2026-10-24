---
title: EVM 06 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 06 - Hasil Praktik

Hasil percobaan untuk materi [EVM 06 - Mekanisme Burn UNI di Fork](../EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md). Dikerjakan 2026-09-24:
rekonstruksi buyback-burn UNI dari transaksi nyata, lalu dijalankan sendiri di `anvil` fork
Ethereum mainnet (blok ~26.046.123, RPC `eth.drpc.org`). Melengkapi
UNI Diperiksa dengan Saringan Lima Chain.

## Transaksi burn asli

Burn terbaru saat itu `0x0bdd8b780b66ae68c52eebb7f4660eaddb63b85265549dae86c41d692c422aaa`
(2026-09-23 20:31 UTC):

| | |
| --- | --- |
| Pengirim | EOA `0xFeeFee6e90b7Bc64f6ADBa1dDb6d5A13c2D169d3` |
| Tujuan | kontrak bot `0x9b715efB4EbDB14862d079A841bA4983FFd69236` (19.767 byte) |
| Calldata | 8.228 byte, selector `0x7e874012` (tidak ada di database publik) |
| **Log / gas** | **256 log, 4.236.883 gas** |

Flash loan WETH dari Morpho Blue `0xbbbbbbbbbb9cc5e90e3b3af64bdaf62c37eeffcb`; UNI yang dibeli
tepat **4.000 UNI**.

## Alamat kontrak

| Kontrak | Alamat |
| --- | --- |
| **Firepit** | `0x0d5cd355e2abeb8fb1552f56c965b867346d6721` |
| **TokenJar** | `0xf38521f130fccf29db1961597bc5d2b60f995f85` |

## Konfigurasi Firepit (2026-09-24)

| Parameter | Nilai |
| --- | --- |
| `RESOURCE` | UNI |
| `RESOURCE_RECIPIENT` | `0x…dEaD` |
| `threshold` | **4.000 UNI** (~$36.960 di $9,24) |
| `nonce` | 1.361 (transaksi di atas = 1.359) |
| `MAX_RELEASE_LENGTH` | 20 aset |
| `owner` | Timelock governance UNI `0x1a9C…35BC` |

## TokenJar di fork

Saldo TokenJar: ETH, WETH, USDC, USDT, WBTC, UNI, DAI — **semua 0**.

Persaingan bot yang menjaga nilai burn ≈ nilai fee menjelaskan kenapa burn on-chain cocok ±3–6%
dengan holder revenue DefiLlama di riset UNI.

## Percobaan di fork

### 1. Nonce salah

`release(1360, …)` → `custom error 0x756688fe` = `cast sig 'InvalidNonce()'`.

### 2. Invariant rusak karena menulis storage

Slot mapping `balances` UNI = 4 ditemukan dengan mencocokkan saldo `0xdead`.

Akun `anvil` #0 `0xf39F…2266` diberi 4.000 UNI lewat `anvil_setStorageAt` → `balanceOf` bilang
4.000, tapi transfer revert `Uni::_moveVotes: vote amount underflows`.

### 3. Burn sendiri dari akun baru

| | |
| --- | --- |
| status | sukses, 64.879 gas, 3 log |
| `0xdead` | **+4.000 UNI** |
| Yang diterima | **0 WETH, 0 ETH** (jar kosong) |
| Firepit nonce | 1.361 → 1.362 |
| Hasil | rugi total ~$36.960 |

Bot asli butuh 4,24 jt gas.

### 4. Aturan mint UNI (impersonate Timelock sebagai `minter`)

| Percobaan | Hasil |
| --- | --- |
| mint 2% + 1 wei | ❌ `Uni::mint: exceeded mint cap` |
| **mint tepat 2% (20 jt)** | ✅ `totalSupply` 1,00 M → **1,02 M** |
| mint lagi 1 UNI | ❌ `minting not allowed yet` — terbuka lagi **2027-09-24** |
| mint dari akun biasa | ❌ `only the minter can mint` |

Batas 2% dihitung dari `totalSupply` = 1 miliar, yang masih termasuk 112 jt di `0xdead`.
Terhadap supply efektif (~887,8 jt), 20 jt = **2,25%**, bukan 2%.

## Angka yang bergerak hari itu

Saldo `0xdead` UNI: 112.091.581 (riset, pagi) → **112.159.581** (fork, sore) = **+68.000 UNI dalam
beberapa jam**.

> ⚠️ **Koreksi Sesi 22 ([EVM 22 - Monitoring](../EVM%2022%20-%20Monitoring.md)):** +68.000 UNI di atas **bukan** dari Firepit mainnet
> (nonce cuma naik 1.359 → 1.361). Sejak 2026-09-23 06:00 UTC: 51 transfer, 112.000 UNI ke `0xdead`, hanya
> 4.000 dari Firepit mainnet. Sisanya **Firepit di L2 (threshold 2.000)** yang mem-bridge UNI ke `0xdead` di L1
> lewat `_afterRelease` — datang dari L1StandardBridge Base/Optimism, L1ERC20Gateway Arbitrum, dan
> gateway/NTT lain.

## Jebakan tooling yang ditemui

Semua jebakan tooling di materi ditemui di sesi ini. Detail RPC: publicnode sempat timeout saat
`anvil` start; `eth.drpc.org` stabil.
