---
title: EVM 10 - Burn _burn vs 0xdead
tags: [learning, evm, solidity, erc20, burn, tokenomics]
updated: 2026-10-24
---

# Burn: `_burn()` vs `0xdead`

Sesi 10 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar dua cara membakar token — `_burn()` yang
mengurangi `totalSupply`, dan transfer ke `0xdead` yang tidak — dan cara menghitung supply efektif
yang benar untuk keduanya. Lanjutan dari [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md).

Hasil percobaan untuk materi ini ada di [EVM 10 - Hasil Praktik](Hasil%20Praktik/EVM%2010%20-%20Hasil%20Praktik.md).

## Satu burn, dua cerita

Contoh: bakar 300.000 dari supply 1.000.000, dengan dua token yang identik kecuali cara burn-nya.

| | Gaya A: `_burn()` | Gaya B: transfer ke `0xdead` |
| --- | ---: | ---: |
| `totalSupply()` | **700.000** | **1.000.000** (tidak berubah) |
| `balanceOf(0xdead)` | 0 | 300.000 |
| Event | `Transfer(from, 0x0, v)` | `Transfer(from, 0xdead, v)` |
| **Supply efektif** = `totalSupply − balanceOf(0xdead)` | 700.000 | 700.000 |

**Burn-nya sama, `totalSupply` bercerita beda.** Kalau hanya membaca `totalSupply`, token gaya B
terlihat tidak pernah membakar apa pun, dan FDV-nya terlihat jauh lebih besar dari kenyataan. Rumus
supply efektif menyamakan keduanya.

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
OpenZeppelin. Token tanpa fungsi `burn()` (misalnya UNI) tidak punya jalan lain selain mengirim ke
alamat yang kuncinya tidak dimiliki siapa pun. `0xdead` adalah EOA biasa dengan nonce 0 (lihat
[EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md)).

## Supply efektif yang benar

```solidity
supply = totalSupply() - balanceOf(0xdead) - Σ balanceOf(alamat yang dikecualikan)
```

Contoh: supply 100, 20 sudah di `0xdead`, 30 lagi antre di Safe burn (menunggu dikirim ke `0xdead`).

| Pembaca | Hasil |
| --- | ---: |
| Cuma kurangi `0xdead` | 80 |
| Kurangi `0xdead` + Safe burn | **50** |

Contoh nyata CAKE ([EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md)): Safe burn `0xceba…` dan stok MasterChefV2 harus
ikut dikurangi.

## Gas

Biaya kedua gaya praktis setara; `_burn()` sedikit lebih murah. Transfer **pertama** ke `0xdead`
lebih mahal (sekitar +17.100 gas) karena membuat slot saldo `0xdead` yang baru (nol → bukan nol).

**Jebakan:** angka gas yang diukur di dalam test Foundry bisa menyesatkan, bahkan urutannya
terbalik dibanding transaksi nyata (juga dengan `vm.cool`). Untuk angka gas, selalu ukur transaksi
nyata, misalnya di `anvil`.

## Peta pola burn

| Pola | `totalSupply` bisa dipercaya? | Cara membaca burn yang benar | Contoh |
| --- | --- | --- | --- |
| A — fungsi `burn()` / `burnFrom()` | ✅ ya | `totalSupply()` atau `getPastTotalSupply()` | ETHFI |
| B — transfer ke `0xdead`, tanpa `burn()` | ❌ tidak pernah turun | `balanceOf(0xdead)` | UNI ([EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)) |
| B + mint-lalu-bakar berkala | ❌ bisa **naik terus** | `totalSupply − 0xdead − alamat burn/stok yang dikecualikan` | CAKE |
| Coin native dikirim ke `0xdead` | tidak ada `totalSupply()` | `eth_getBalance(0xdead)` | BNB di BSC |

**Aturan umum:** sebelum mengukur burn token apa pun, cek dulu **kontraknya punya fungsi `burn`
atau tidak**, lalu cek saldo `0xdead` dan `0x0`. Dua pertanyaan itu menentukan metode mana yang
valid. Memakai metode gaya A untuk token gaya B (atau sebaliknya) menghasilkan angka burn yang
salah.

## Coba sendiri

1. Tulis dua token: `SupplyBurnToken` (gaya A) dan `DeadBurnToken` (gaya B), plus `SupplyReader`
   yang menghitung supply efektif.
2. Bakar 300.000 dari 1.000.000 di keduanya, lalu bandingkan `totalSupply` dan `balanceOf(0xdead)`.
3. Tambahkan alamat Safe burn, dan pastikan pembaca supply ikut mengurangi alamat itu.
4. Deploy ke `anvil`, lalu ukur gas `_burn()`, transfer pertama ke `0xdead`, dan transfer
   berikutnya dari receipt transaksi nyata.

```bash
forge test --match-path test/BurnStyles.t.sol -vv
cast call <token> 'totalSupply()(uint256)'
cast call <token> 'balanceOf(address)(uint256)' 0x000000000000000000000000000000000000dEaD
cast send <token> 'burn(uint256)' <amount> --private-key <pk> --json | jq .gasUsed
```
