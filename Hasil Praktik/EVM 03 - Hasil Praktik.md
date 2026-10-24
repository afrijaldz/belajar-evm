---
title: EVM 03 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 03 - Hasil Praktik

Hasil percobaan untuk materi [EVM 03 - Transaksi dan Gas](../EVM%2003%20-%20Transaksi%20dan%20Gas.md). Dijalankan 2026-09-24 di `anvil`
lokal (Foundry 1.7.1, hardfork `Bpo1`, block gas limit sengaja dikecilkan ke 100.000) dan di
Ethereum/BSC mainnet. Project: `~/Documents/riset/evm/account-model` (kontrak `Tally` dengan
event + revert).

## Transaksi mentah

Transfer 1 ETH tipe EIP-1559 yang dibuat dengan `cast mktx` = **119 byte**, byte pertama `0x02`.

| Field | Nilai di percobaan |
| --- | --- |
| `chainId` | 31337 |
| `nonce` | 0 |
| `gas` | 21.000 |
| `maxFeePerGas` | 3 gwei |
| `maxPriorityFeePerGas` | 2 gwei |
| `to`, `value`, `input` | B, 1 ETH, `0x` |
| `r`, `s`, `yParity` | … |

## EIP-1559

- Transfer di atas, base fee blok 1 gwei, tip 2 gwei: base 21.000 gwei dibakar, tip 42.000 gwei
  ke coinbase (di `anvil` = `0x0`), total 63.000 gwei.
- Dengan maxFee 100 gwei dan tip 2 gwei, yang dibayar tetap base + tip = 0,9275 + 2 =
  **2,9275 gwei**.
- maxFee di bawah base fee ditolak dengan pesan `max fee per gas less than block base fee`.

## Base fee

| Kondisi blok sebelumnya | Hasil terukur | Prediksi rumus |
| --- | --- | --- |
| 21.000 dari target 50.000 | 1 → **0,9275** gwei | 0,9275 ✓ |
| Kosong (3 blok berturut) | ×**0,875** tiap blok (0,6957 → 0,6087 → 0,5326) | turun maks −12,5% ✓ |
| 99.000 dari limit 100.000 | ×**1,1225** (466.036.724 → 523.126.222 wei) | 523.126.223 (selisih 1 wei) ✓ |

## Calldata

| Calldata | gasUsed | Per byte |
| --- | ---: | ---: |
| kosong | 21.000 | — |
| 100 × `0x00` | 22.000 | **10** |
| 100 × `0xff` | 25.000 | **40** |

## Storage (`Tally.set(uint256)`, slot yang sama)

| Operasi | gasUsed | Selisih dari "ubah nilai" |
| --- | ---: | ---: |
| **0 → 42** (slot baru) | 43.740 | **+17.100** (20.000 vs 2.900) |
| 42 → 43 (ubah) | 26.640 | baseline |
| 43 → 43 (sama) | 23.840 | −2.800 |
| 43 → 0 (hapus) | 21.828 | −4.812 (**refund 4.800** + 12 gas calldata nol) |

## Revert dan out of gas

| Kasus | status | gasUsed | Nonce | State |
| --- | --- | --- | --- | --- |
| Revert (`add(0)`, `require` gagal) | `false` | 21.875 | naik | tidak berubah |
| Out of gas (`add(5)`, limit 23.000) | `false` | **23.000 = seluruh limit** | naik | tidak berubah |

`cast run <tx>` untuk transaksi revert menampilkan `[Revert] amount is zero`.

## Log dari `Tally.add(5)`

`add(5)` sukses memancarkan `Added(address indexed by, uint256 amount, uint256 newTotal)`:

| Bagian log | Isi |
| --- | --- |
| `address` | Tally |
| `topics[0]` | `0x2a8f5e2f…` = `keccak256("Added(address,uint256,uint256)")` |
| `topics[1]` | `0x…f39fd6e5…` (`by` = akun #0 `anvil`) |
| `data` | `5`, `5` |

Topic `0xddf252ad…` (`Transfer`) dan filter `[T, null, DEAD]` adalah yang dipakai di riset
tokenomics untuk menarik burn CAKE dan UNI lewat `eth_getLogs`.

## Chain nyata (2026-09-24)

| | Ethereum | BSC |
| --- | ---: | ---: |
| Block gas limit | 60.000.000 | 70.000.000 |
| Terisi | 65,2% | 25,5% |
| **Base fee** | 0,108 gwei | **0** |
| `eth_gasPrice` | 0,108 gwei | 0,05 gwei |
| Transaksi per blok | 234 | 121 (blok ~27x lebih sering: 0,45 detik vs 12 detik) |

- Sumber burn BSC (BEP-95 dan auto-burn kuartalan): lihat BNB Diperiksa Ulang 2026-09-24.
- **Transaksi mint-burn CAKE** `0xb7a7e298…abd11` (CAKE Diperiksa Ulang 2026-09-24): tipe
  `0x0` (legacy), **2.454.872 gas**, 0,1 gwei, fee **0,000245 BNB (~$0,19)**, 17 log
  (7× `UpdatePool`, 4× `Transfer`, dll.).

## Jebakan tooling yang ditemui

Semua jebakan di bagian "Jebakan" materi ditemui di sesi ini. Satu lagi yang khusus mesin ini:
fungsi shell bernama `g` bentrok dengan alias git di zsh.
