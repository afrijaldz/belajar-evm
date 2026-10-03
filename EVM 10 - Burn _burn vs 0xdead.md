---
title: EVM 10 - Burn _burn vs 0xdead
tags: [learning, evm, solidity, erc20, burn, tokenomics]
updated: 2026-09-24
---

# Burn: `_burn()` vs `0xdead`

Catatan dari Sesi 10 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Dua token identik yang cuma beda cara burn,
diuji dengan Foundry (4 test) dan diukur gas-nya di `anvil`. Lanjutan dari
[EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md). Ini sisi "menulis" dari temuan riset tokenomics.

Project: `~/Documents/riset/evm/solidity-basics` — `src/BurnStyles.sol` (`SupplyBurnToken`,
`DeadBurnToken`, `SupplyReader`), `test/BurnStyles.t.sol`. Token latihan `MyToken` sengaja tidak
disentuh — latihan `burn` Sesi 8 masih untukmu.

## Satu burn, dua cerita

Bakar 300.000 dari supply 1.000.000:

| | Gaya A: `_burn()` | Gaya B: transfer ke `0xdead` |
| --- | ---: | ---: |
| `totalSupply()` | **700.000** | **1.000.000** (tidak berubah) |
| `balanceOf(0xdead)` | 0 | 300.000 |
| Event | `Transfer(from, 0x0, v)` | `Transfer(from, 0xdead, v)` |
| **Supply efektif** = `totalSupply − balanceOf(0xdead)` | 700.000 | 700.000 |

**Burn-nya sama, `totalSupply` bercerita beda.** Akar kesalahan riset: UNI tercatat "nol burn"
padahal 112 jt dibakar; FDV CAKE terlihat 15,4x lebih besar. Rumus supply efektif menyamakan
keduanya.

```solidity
// Style A
function burn(uint256 value) external {
    balanceOf[msg.sender] -= value;
    totalSupply -= value;
    emit Transfer(msg.sender, address(0), value);
}
// Style B: tidak ada fungsi burn. User memanggil transfer(0xdead, value).
```

## Kenapa `0xdead` ada

`transfer(address(0), …)` **ditolak** (`transfer to zero address`) — aturan yang sama di
OpenZeppelin. Token tanpa fungsi `burn()` (UNI) tidak punya jalan lain selain mengirim ke alamat
yang kuncinya tidak dimiliki siapa pun. `0xdead` adalah EOA biasa dengan nonce 0 (lihat
[EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md)).

## Supply efektif yang benar

```solidity
supply = totalSupply() - balanceOf(0xdead) - Σ balanceOf(alamat yang dikecualikan)
```

Test `test_BurnSafeMustAlsoBeExcluded`: 20 sudah di `0xdead`, 30 antre di Safe burn.

| Pembaca | Hasil |
| --- | ---: |
| Cuma kurangi `0xdead` | 80 |
| Kurangi `0xdead` + Safe burn | **50** |

Pelajaran CAKE Sesi 5 ([EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)): Safe burn `0xceba…` dan stok MasterChefV2 harus
ikut dikurangi.

## Gas (anvil, transaksi terpisah)

| | Gas |
| --- | ---: |
| `_burn()` (setiap kali) | **34.127** |
| → `0xdead` pertama | 51.935 (slot saldo `0xdead` baru, +17.100) |
| → `0xdead` berikutnya | 34.835 |

Praktis setara; `_burn()` ~700 gas lebih murah. **Angka di dalam test Foundry justru terbalik**
(18.224 vs 14.539, bahkan dengan `vm.cool`) — test gas itu dihapus karena menyesatkan. Untuk angka
gas, selalu ukur transaksi nyata.

## Peta semua pola burn dari riset

| Token | Pola | `totalSupply` bisa dipercaya? | Cara membaca burn yang benar |
| --- | --- | --- | --- |
| **ETHFI** | A — `burn()` / `burnFrom()` | ✅ ya | `totalSupply()` atau `getPastTotalSupply()` (ETHFI Diperiksa dengan Saringan Lima Chain) |
| **UNI** | B — Firepit → `0xdead` (tidak ada `burn()`) | ❌ selalu 1 M | `balanceOf(0xdead)` ([EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) |
| **CAKE** | B + mint-lalu-bakar mingguan | ❌ **naik terus** (2,44 M dalam 10 bulan) | `totalSupply − 0xdead − Safe burn − MasterChefV2` (CAKE Diperiksa Ulang 2026-09-24) |
| **BNB** | coin native → `0xdead` di BSC | tidak ada `totalSupply()` | `eth_getBalance(0xdead)` (BNB Diperiksa Ulang 2026-09-24) |
| **HYPE** | native HyperCore/HyperEVM, supply berkurang langsung | ✅ dari API `tokenDetails` | `1 M − totalSupply`; alamat burn cuma 1.676 (HYPE Diperiksa Ulang 2026-09-24) |
| RAY, PUMP, ORCA (Solana) | SPL `burn` mengurangi supply langsung (setara A) | ✅ `getTokenSupply` | tidak ada pola dead address di Solana |

**Aturan umum:** sebelum mengukur burn token apa pun, cek dulu **kontraknya punya fungsi `burn`
atau tidak**, lalu cek saldo `0xdead` dan `0x0`. Dua pertanyaan itu menentukan metode mana yang
valid. Kesalahan di riset (UNI "nol burn", HYPE "1.676 dibakar") terjadi karena memakai metode
gaya A untuk token gaya B, dan sebaliknya.
