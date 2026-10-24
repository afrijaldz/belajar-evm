---
title: EVM 01.09 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 01.09 - Hasil Praktik

Hasil percobaan untuk materi [EVM 01.09 - Bytecode yang Sama](../EVM%2001%20-%20Detail/EVM%2001.09%20-%20Bytecode%20yang%20Sama.md). Dijalankan 2026-10-03 di
Ethereum mainnet dan BSC mainnet (baca saja).

## Multicall3 di dua chain

Kontrak Multicall3 di `0xcA11bde05977b3631167028862bE2a173976CA11` punya hash bytecode
identik di Ethereum dan BSC:

```text
0xd5c15df687b16f2ff992fc8d767b4216323184a2bbc6ee2f9c398c318e770891
```

Perintah: `cast code <addr> --rpc-url <rpc> | cast keccak`.
