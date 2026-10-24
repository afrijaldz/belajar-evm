---
title: EVM 18 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 18 - Hasil Praktik

Hasil percobaan untuk materi [EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md). Dijalankan 2026-09-24: serangan oracle dengan flash
loan Morpho Blue asli di fork Ethereum (blok ~26.046.522), perbaikan dengan TWAP, dan jebakan `tx.origin`.
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

- Pompa lalu buang 1.000 ETH cuma rugi ~0,55 ETH (fee dua arah).
- `update()` dipanggil tepat setelah pompa: TWAP tetap 99, spot 12.066.
- Koreksi asumsi: awalnya serangan ke lender TWAP diharapkan revert, ternyata **tidak revert** — biaya
  round-trip kecil tertutup pinjaman wajar. Test diubah untuk menguji bahwa lender tidak kehilangan lebih
  dari batas wajar dan penyerang berakhir rugi.

## `tx.origin`

| Dompet | Pemilik memanggil `FakeAirdrop.claim()` |
| --- | --- |
| `require(tx.origin == owner)` | **terkuras 10 ETH** |
| `require(msg.sender == owner)` | revert `not owner` |
