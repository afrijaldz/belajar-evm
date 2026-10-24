---
title: EVM 17.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 17.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 17.01 - delegatecall](../EVM%2017%20-%20Detail/EVM%2017.01%20-%20delegatecall.md). File: `~/Documents/riset/evm/proxy/src/Delegatecall.sol`
dan `test/Delegatecall.t.sol`. `Probe.record(n)` menyimpan `number`, `msg.sender`, `msg.value`, dan
`address(this)`. `Caller` punya layout storage yang sama dan memanggil `record` dengan dua cara.

```bash
cd ~/Documents/riset/evm/proxy && forge test --match-path test/Delegatecall.t.sol -vv
```

Hasil: **5 passed, 0 failed**.

## `call` vs `delegatecall`

Alice memanggil `Caller` dengan 1 ETH dan `n = 7`:

| Hasil | `viaCall` | `viaDelegatecall` |
| --- | --- | --- |
| Storage yang berubah | `Probe` (`number` = 7) | **`Caller`** (`number` = 7) |
| Storage yang tetap 0 | `Caller` | `Probe` |
| `sender` yang tercatat | `Caller` | **alice** |
| `value` yang tercatat | 1 ETH | 1 ETH (`msg.value` tx asli) |
| `self` (`address(this)`) | `Probe` | **`Caller`** |
| ETH 1 ETH berakhir di | `Probe` | `Caller` |

## Test lain

| Test | Hasil |
| --- | --- |
| `delegatecall` ke alamat **tanpa kode** | `ok = true`, return data 0 byte, tidak ada yang berubah |
| `delegatecall` ke fungsi yang revert | `ok = false`, return data = `Error(string)` "probe failed"; revert tidak diteruskan otomatis |
| `UnsafeWallet.execute(target, data)` yang memakai `delegatecall`, dipanggil attacker dengan `TakeOver.pwn()` | `owner` wallet berubah menjadi **attacker**; storage `TakeOver` sendiri tetap kosong |

## Opcode

Disassembly `Caller` berisi 2 `DELEGATECALL` (`viaDelegatecall` dan `rawDelegatecall`) dan 1 `CALL`
(`viaCall`).

Angka insiden Parity (~513.000 ETH) diambil dari laporan publik dan tidak dicek ulang di sesi ini.
