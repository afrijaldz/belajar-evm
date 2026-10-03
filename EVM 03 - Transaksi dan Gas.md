---
title: EVM 03 - Transaksi dan Gas
tags: [learning, evm, ethereum, bsc, gas]
updated: 2026-09-24
---

# Transaksi dan Gas EVM

Catatan dari Sesi 3 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Semua angka hasil observasi langsung di
`anvil` lokal (Foundry 1.7.1, hardfork `Bpo1`, block gas limit sengaja dikecilkan ke 100.000)
dan di Ethereum/BSC mainnet. Lanjutan dari [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md).

Project praktik: `~/Documents/riset/evm/account-model` (kontrak `Tally` dengan event + revert).

## Transaksi = 119 byte data yang ditandatangani

`cast mktx` membuat dan menandatangani transaksi **tanpa mengirimnya**. Transfer 1 ETH tipe
EIP-1559 = **119 byte**, byte pertama `0x02` = tipe transaksi.

| Field | Contoh | Arti |
| --- | --- | --- |
| `chainId` | 31337 | mencegah replay di chain lain |
| `nonce` | 0 | urutan transaksi pengirim |
| `gas` | 21.000 | **batas** gas |
| `maxFeePerGas` | 3 gwei | harga maksimal per gas |
| `maxPriorityFeePerGas` | 2 gwei | tip maksimal untuk validator |
| `to`, `value`, `input` | B, 1 ETH, `0x` | tujuan, jumlah, calldata |
| `r`, `s`, `yParity` | … | tanda tangan |

**Tidak ada field `from`.** Pengirim dihitung ulang dari tanda tangan — tidak bisa dipalsukan.

## EIP-1559: ke mana biaya pergi

Transfer di atas, base fee blok 1 gwei, tip 2 gwei:

| Bagian | Rumus | Jumlah | Ke mana |
| --- | --- | ---: | --- |
| Base fee | 1 gwei × 21.000 | 21.000 gwei | **dibakar** |
| Priority fee | 2 gwei × 21.000 | 42.000 gwei | validator (coinbase; di `anvil` = `0x0`) |
| **Total** | 3 gwei × 21.000 | **63.000 gwei** | |

- **`maxFeePerGas` cuma plafon.** Dengan maxFee 100 gwei dan tip 2 gwei, yang dibayar tetap
  base + tip = 0,9275 + 2 = **2,9275 gwei**.
- **maxFee di bawah base fee langsung ditolak** (`max fee per gas less than block base fee`),
  tidak masuk blok.

Ini mekanisme burn yang sama dengan di riset tokenomics — burn di level protokol, bukan dari
kontrak.

## Base fee diatur rumus, bukan lelang

```
base_baru = base_lama × (1 + 1/8 × (gas_terpakai − target) / target)      target = gasLimit / 2
```

| Kondisi blok sebelumnya | Hasil terukur | Prediksi |
| --- | --- | --- |
| 21.000 dari target 50.000 | 1 → **0,9275** gwei | 0,9275 ✓ |
| Kosong (3 blok berturut) | ×**0,875** tiap blok (0,6957 → 0,6087 → 0,5326) | turun maks −12,5% ✓ |
| 99.000 dari limit 100.000 | ×**1,1225** (466.036.724 → 523.126.222 wei) | 523.126.223 (selisih 1 wei) ✓ |

Transaksi yang butuh gas melebihi block gas limit ditolak saat estimasi.

## Harga calldata: 10 dan 40 gas per byte

| Calldata | gasUsed | Per byte |
| --- | ---: | ---: |
| kosong | 21.000 | — |
| 100 × `0x00` | 22.000 | **10** |
| 100 × `0xff` | 25.000 | **40** |

Bukan 4 dan 16 seperti di banyak tutorial lama. Untuk transaksi yang dominan data, berlaku
**harga minimum EIP-7623** (Pectra): 10 gas per "token", byte nol = 1 token, byte bukan nol =
4 token.

## Storage adalah operasi termahal

`Tally.set(uint256)`, slot yang sama:

| Operasi | gasUsed | Selisih dari "ubah nilai" |
| --- | ---: | ---: |
| **0 → 42** (slot baru) | 43.740 | **+17.100** (20.000 vs 2.900) |
| 42 → 43 (ubah) | 26.640 | baseline |
| 43 → 43 (sama) | 23.840 | −2.800 |
| 43 → 0 (hapus) | 21.828 | −4.812 (**refund 4.800** + 12 gas calldata nol) |

Membuat satu slot baru (20.000) ≈ satu transfer ETH penuh. Itu sebabnya transfer token ke
alamat yang belum pernah punya saldo lebih mahal.

## Gagal tetap bayar

| Kasus | status | gasUsed | Nonce | State |
| --- | --- | --- | --- | --- |
| Revert (`add(0)`, `require` gagal) | `false` | 21.875 | naik | tidak berubah |
| Out of gas (`add(5)`, limit 23.000) | `false` | **23.000 = seluruh limit** | naik | tidak berubah |

- Revert: yang dibayar hanya gas yang **terpakai** sampai titik revert.
- Out of gas: **seluruh gas limit hangus**.
- Alasan revert bisa dibaca ulang: `cast run <tx>` → `[Revert] amount is zero`.

## Receipt dan log

`add(5)` sukses memancarkan `Added(address indexed by, uint256 amount, uint256 newTotal)`:

| Bagian log | Isi | Arti |
| --- | --- | --- |
| `address` | Tally | kontrak yang memancarkan |
| `topics[0]` | `0x2a8f5e2f…` | `keccak256("Added(address,uint256,uint256)")` |
| `topics[1]` | `0x…f39fd6e5…` | parameter `indexed` (`by`) — bisa difilter |
| `data` | `5`, `5` | parameter non-indexed |

**Ini yang dipakai di riset tokenomics.** `0xddf252ad…` di filter `eth_getLogs` untuk burn CAKE
dan UNI = `keccak256("Transfer(address,address,uint256)")`. Filter `[T, null, DEAD]` = "Transfer,
pengirim siapa saja, penerima `0xdead`". Bisa karena `from` dan `to` bertipe `indexed`.

## Chain nyata (2026-09-24)

| | Ethereum | BSC |
| --- | ---: | ---: |
| Block gas limit | 60.000.000 | 70.000.000 |
| Terisi | 65,2% | 25,5% |
| **Base fee** | 0,108 gwei | **0** |
| `eth_gasPrice` | 0,108 gwei | 0,05 gwei |
| Transaksi per blok | 234 | 121 (blok ~27x lebih sering: 0,45 detik vs 12 detik) |

- **Base fee BSC selalu 0.** Format EIP-1559 dipakai, tapi base fee tidak dibakar. Burn BSC
  datang dari BEP-95 dan auto-burn kuartalan — lihat BNB Diperiksa Ulang 2026-09-24.
- **Transaksi mint-burn CAKE** `0xb7a7e298…abd11` (CAKE Diperiksa Ulang 2026-09-24):
  tipe `0x0` (legacy), **2.454.872 gas**, 0,1 gwei, fee **0,000245 BNB (~$0,19)**, 17 log
  (7× `UpdatePool`, 4× `Transfer`, dll.).

## Jebakan tooling yang ditemui

- `cast block <n> gasLimit` salah — yang benar `cast block <n> --field gasLimit`.
- `jq tonumber` tidak bisa membaca hex (`0x5208`). Pakai Python `int(x, 16)` atau `cast to-dec`.
- Aritmetika zsh `$(( ))` overflow di atas 2^63 — saldo dalam wei (≈10^22) jadi salah. Pakai Python.
- Fungsi shell bernama `g` bentrok dengan alias git di zsh.
- Transaksi revert tidak bisa dikirim lewat `cast send` biasa karena estimasi gas gagal duluan.
  Pasang `--gas-limit` manual supaya tetap terkirim.

## Perintah

```bash
anvil --port 8545 --gas-limit 100000                     # block kecil supaya base fee mudah bergerak
cast mktx <to> --value 1ether --private-key <pk> --nonce 0 --gas-limit 21000 \
  --priority-gas-price 2gwei --gas-price 3gwei --chain 31337
cast decode-transaction <raw>
cast publish <raw>
cast receipt <tx> gasUsed ; cast receipt <tx> effectiveGasPrice
cast block latest --field baseFeePerGas
cast rpc anvil_mine 1                                    # blok kosong
cast rpc anvil_setBlockGasLimit 30000000
cast run <tx>                                            # trace + alasan revert
cast keccak 'Transfer(address,address,uint256)'          # topic[0] event
```
