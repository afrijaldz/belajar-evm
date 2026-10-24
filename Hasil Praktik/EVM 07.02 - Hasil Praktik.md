---
title: EVM 07.02 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 07.02 - Hasil Praktik

Hasil percobaan untuk materi [EVM 07.02 - Fallback](../EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md). Test Foundry di project
`~/Documents/riset/evm/solidity-basics`. File: `src/Fallback.sol` dan `test/Fallback.t.sol`.
Pengecekan USDC di Ethereum mainnet dilakukan 2026-10-03.

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-path test/Fallback.t.sol -vv
```

Hasil: **6 passed, 0 failed**.

| Test | Hasil |
| --- | --- |
| Kirim 1 ETH tanpa calldata ke kontrak yang hanya punya `fallback() payable` | `fallback()` jalan, `msg.data.length` 0, `msg.value` 1 ETH |
| Panggil dengan calldata 2 byte (`0xabcd`) | `fallback()` jalan, `msg.data.length` **2** |
| Panggil `transfer(address,uint256)` ke kontrak yang tidak punya fungsi itu | `fallback()` jalan dan melihat calldata utuh, **68 byte** (4 selector + 32 + 32) |
| `EchoFallback` (bentuk 2) dipanggil dengan `0xdeadbeef01` | return data persis `0xdeadbeef01`, tanpa ABI encoding |
| `StrictFallback` dipanggil dengan `withdraw()` | revert `UnknownCall(0x3ccfd60b)` (selector `withdraw()`); fungsi `known()` tetap jalan normal |
| `MiniProxy` dipanggil `increment()` 2 kali | `number` di proxy = **2**, di kontrak logika tetap **0**; slot 0 proxy = 2 |

## Bytecode

| Kontrak | Kalau tidak ada selector yang cocok |
| --- | --- |
| `OnlyPayableFunction` (tanpa `fallback()`) | lompat ke `PUSH0 PUSH0 REVERT` |
| `FallbackOnly` (dengan `fallback()`) | lompat ke `0x37`, yaitu badan `fallback()` (langsung `SLOAD` untuk `hits++`) |

## USDC di Ethereum mainnet (2026-10-03)

USDC `0xA0b8…eB48`:

| Cek | Hasil |
| --- | --- |
| Ukuran bytecode proxy | 2.186 byte |
| Selector `symbol()` `0x95d89b41` ada di bytecode proxy? | **tidak** |
| `cast call <USDC> "symbol()(string)"` | `"USDC"` |
| Selector `upgradeTo`, `implementation`, `admin` di bytecode proxy | ada (fungsi milik proxy sendiri) |
| Slot ZeppelinOS implementation `0x7050…f8c3` | `0x43506849d7c04f9138d1a2050bbf3a0c054402dd` |

Panggilan `symbol()` diteruskan ke kontrak logika `0x4350…02dd`.
