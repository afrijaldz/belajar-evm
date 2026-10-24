---
title: EVM 24 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 24 - Hasil Praktik

Hasil percobaan untuk materi [EVM 24 - Capstone SupplyWatch](../EVM%2024%20-%20Capstone%20SupplyWatch.md). Dijalankan 2026-09-24. Project:
`~/Documents/riset/evm/capstone` — `src/SupplyWatch.sol`, `test/SupplyWatch.t.sol` (6 unit + 2 fork),
`script/DeploySupplyWatch.s.sol`.

```bash
cd ~/Documents/riset/evm/capstone
forge test --match-contract SupplyWatchUnitTest -vv
forge test --match-test test_UniLast30Days -vv
forge test --match-test test_CakeLast12Months --compute-units-per-second 20 -vv
```

Latar belakang: angka agregator yang salah ditemui di riset sebelumnya — CoinGecko di UNI, HYPE, ETHFI.

## Unit test

6 unit test, semua lolos.

## Fork dengan token asli

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

| Chain | Alamat (simulasi, tidak di-broadcast) |
| --- | --- |
| prediksi `cast create2` | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Ethereum (1) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Base (8453) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |
| Arbitrum (42161) | `0xEF4833b7FEF59289C4B96249A7F4975a122a5B3F` |

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
- drpc gratis bisa `eth_call` historis tapi gagal/timeout saat fork membaca akun & storage lama (HTTP 408,
  500). Tenderly (`mainnet.gateway.tenderly.co`) lolos keduanya.
- `sed` dengan pemisah `#` gagal kalau teks penggantinya berisi `#`.
- Alamat test contract Foundry `0x5615…b72f` punya saldo di mainnet.

## Belum dikerjakan

Deploy sungguhan butuh saldo: dijalankan bersama Sesi 21 setelah faucet masuk.
