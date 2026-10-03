---
title: EVM 18 - Security Basics
tags: [learning, evm, security, oracle, flash-loan, audit]
updated: 2026-09-24
---

# Security Basics

Catatan dari Sesi 18 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24 — **penutup Fase 3**. Serangan oracle dengan flash
loan Morpho Blue asli di fork Ethereum (blok ~26.046.522), perbaikannya dengan TWAP, jebakan `tx.origin`,
dan checklist auditor dari seluruh sesi. Lanjutan dari [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md).

Project: `~/Documents/riset/evm/security` — `src/Lending.sol` (`SpotPriceSource`, `TwapPriceSource`,
`Lender`), `src/Attacker.sol`, `src/TxOrigin.sol`, `test/Oracle.t.sol`, `test/TxOrigin.t.sol`.

```bash
cd ~/Documents/riset/evm/security
forge test --match-contract OracleTest --fork-url https://eth.drpc.org --fork-block-number $(cast block-number --rpc-url https://ethereum-rpc.publicnode.com) -vv
forge test --match-contract TxOriginTest -vv
```

## Manipulasi oracle dengan flash loan

Setup: GOV + pool Uniswap v2 (1.000.000 GOV + 100 ETH, 1 GOV = 100 micro-ETH). `Lender` memegang 200 WETH,
meminjamkan sampai 50% nilai jaminan GOV.

Serangan dalam **satu transaksi** (`OracleAttacker`):

1. Flash loan **1.000 WETH** dari Morpho Blue `0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb` (fee 0; Morpho
   memegang 13.272 WETH).
2. Pompa: beli GOV dengan 1.000 WETH → harga spot **100 → 12.066 micro-ETH (×121)**.
3. Setor 40.000 GOV (~4 ETH di harga wajar) sebagai jaminan, pinjam semua WETH.
4. Buang GOV hasil pompa kembali ke pool → dapat **~999,45 WETH**.
5. Morpho menarik 1.000 WETH kembali.

| | Oracle **spot** (`getReserves`) | Oracle **TWAP** 1 jam |
| --- | ---: | ---: |
| Harga yang dipakai saat serangan | ×121 | wajar (99 micro-ETH) |
| WETH lender tersisa | **0 / 200** | **198 / 200** (hanya pinjaman wajar 2 ETH) |
| Penyerang | **+199 ETH** | terima 1,45 ETH, tinggalkan jaminan 4 ETH → **−2,5 ETH** |

### Temuan penting

- **Memanipulasi spot di pool dangkal nyaris gratis.** Pompa lalu buang 1.000 ETH cuma rugi ~0,55 ETH
  (fee dua arah); sebagian besar price impact kembali saat dump. Biaya manipulasi = fee, bukan ukuran
  modal — dan modalnya dipinjam gratis.
- **TWAP menghitung harga × detik harga itu bertahan.** Harga yang dipompa dan dibuang dalam satu transaksi
  bertahan **0 detik**. Bahkan `update()` dipanggil tepat setelah pompa: TWAP tetap 99, spot 12.066.
- Koreksi asumsi: serangan ke lender TWAP **tidak revert** — biaya round-trip kecil tertutup pinjaman
  wajar. Yang benar diuji: lender tidak kehilangan lebih dari batas wajar, penyerang berakhir rugi.

### Cara TWAP Uniswap v2 bekerja

Pair menyimpan `price0CumulativeLast` / `price1CumulativeLast` = Σ(harga × detik), format UQ112x112.

```
harga rata-rata = (kumulatif_sekarang − kumulatif_lalu) / detik_berlalu
kumulatif_sekarang = kumulatif_tersimpan + harga_reserve_sekarang × (now − blockTimestampLast)
```

Overflow kumulatif disengaja (`unchecked`), sama seperti Uniswap. Kelemahan TWAP: lambat mengikuti harga
nyata, dan pool yang sangat dangkal tetap bisa dimanipulasi kalau penyerang sanggup menahan harga beberapa
blok.

## Access control: `tx.origin`

| Dompet | Pemilik memanggil `FakeAirdrop.claim()` |
| --- | --- |
| `require(tx.origin == owner)` | **terkuras 10 ETH** — di panggilan bersarang `tx.origin` tetap pemilik |
| `require(msg.sender == owner)` | revert `not owner` — `msg.sender` = kontrak airdrop |

Jangan pernah pakai `tx.origin` untuk otorisasi.

## Checklist auditor (Sesi 7–18)

| # | Pertanyaan | Kalau salah | Sesi |
| ---: | --- | --- | --- |
| 1 | Aritmetika `unchecked` benar-benar tidak bisa overflow? Pembulatan ke arah yang aman? | saldo memutar / bocor | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md) |
| 2 | Struct diubah lewat `storage`, bukan `memory`? | perubahan hilang diam-diam | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md) |
| 3 | `from == to` sudah diuji? | token tercetak dari udara | [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md) |
| 4 | Approval: race, unlimited, permit phishing | dana terkuras tanpa transaksi korban | [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) |
| 5 | `totalSupply` jujur? Ada saldo di `0xdead`/Safe burn? | supply salah baca | [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md) |
| 6 | Event lengkap untuk setiap perubahan state penting | tidak bisa diaudit / di-index | [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) |
| 7 | **CEI**: state diubah sebelum panggilan eksternal? | reentrancy | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) |
| 8 | Kredit jumlah yang **diterima**, bukan yang diminta? | insolvent dengan fee token | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) |
| 9 | Token tanpa `bool` / alamat tanpa kode ditangani? | USDT tidak bisa dipakai / "sukses" palsu | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) |
| 10 | Swap: `amountOutMin` dari off-chain + `deadline`? | sandwich −30% | [EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md) |
| 11 | **Harga dari mana?** Spot AMM = bisa dipompa satu transaksi | lender terkuras | catatan ini |
| 12 | Otorisasi pakai `msg.sender`, bukan `tx.origin`; semua fungsi admin dilindungi | dompet/protokol diambil alih | catatan ini |
| 13 | Proxy: slot EIP-1967, storage append-only, initializer terkunci, UUPS menyertakan `upgradeTo` | state rusak / takeover / beku | [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md) |
| 14 | Invariant ditulis, `fail_on_revert` aktif, dibuktikan dengan mutation test | test lolos tapi tidak menguji apa-apa | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) |
| 15 | Siapa yang bisa mengubah apa? (owner, Timelock, multisig, EOA admin) | kepercayaan tersembunyi | [EVM 04 - ABI](EVM%2004%20-%20ABI.md), [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md) |

Pola yang berulang di hampir semua bug: **kontrak mempercayai sesuatu yang bisa dikendalikan pihak lain**
— harga spot, `amount` yang diminta, return value token, `tx.origin`, urutan storage, atau keeper.
Pertanyaan pertama auditor selalu: *angka ini datang dari mana, dan siapa yang bisa menggesernya?*
