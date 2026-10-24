---
title: EVM 07.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 07.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 07.01 - Payable dan receive](../EVM%2007%20-%20Detail/EVM%2007.01%20-%20Payable%20dan%20receive.md). Test Foundry (Solc 0.8.28) di
project `~/Documents/riset/evm/solidity-basics`. File: `src/Payable.sol` dan `test/Payable.t.sol`.

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-path test/Payable.t.sol -vv
```

Hasil: **9 passed, 0 failed**.

| Test | Hasil |
| --- | --- |
| `pay{value: 1 ether}()` (payable) | sukses, `msg.value` = 1 ETH, saldo kontrak 1 ETH |
| `notPayable()` + 1 ETH | **revert**, revert data **0 byte** (tidak ada pesan error) |
| `notPayable()` tanpa ETH | sukses |
| Kirim ETH polos ke kontrak tanpa `receive()`/`fallback()` | **gagal**, saldo kontrak tetap 0 |
| Kirim ETH polos ke kontrak yang punya `receive()` | `receive()` jalan |
| Panggil dengan calldata kosong dan **0 ETH** | `receive()` tetap jalan |
| Panggil `doesNotExist()` + ETH | `fallback()` jalan |
| Kirim ETH dengan calldata 1 byte (`0x01`) | `fallback()` jalan, bukan `receive()` |
| Panggil `hello()` (fungsi yang ada) | `hello()` jalan, `receive()`/`fallback()` tidak disentuh |
| `fallback()` non-payable: panggilan tanpa ETH | sukses |
| `fallback()` non-payable: dengan ETH | **revert** (calldata kosong maupun berisi) |

Baris "calldata kosong dan 0 ETH" yang membuktikan `receive()` dipilih berdasarkan calldata
kosong, bukan berdasarkan ETH.

## `transfer` / `send` / `call`

| Penerima | `transfer` | `send` | `call` |
| --- | --- | --- | --- |
| `LoggingReceiver` (`receive()` hanya `emit`) | sukses | — (tidak diuji) | — (tidak diuji) |
| `CountingReceiver` (`receive()` menulis storage, `count++`) | **revert** | **`false`** | sukses, `count` = 1 |

## ETH paksa lewat `selfdestruct`

Test `test_SelfdestructBypassesReceive`: kontrak `ForceSender` memanggil `selfdestruct(target)`
ke kontrak yang tidak punya `receive()` maupun `fallback()`. Hasilnya, saldo target menjadi
**1 ETH** dan tidak ada kode target yang berjalan (`lastValue` tetap 0).
