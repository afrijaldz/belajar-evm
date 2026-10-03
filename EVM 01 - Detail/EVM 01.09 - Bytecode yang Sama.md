---
title: EVM 01.09 - Bytecode yang Sama
tags: [learning, evm, bytecode]
updated: 2026-10-03
---

# Bytecode yang Sama

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.08 - Gas](EVM%2001.08%20-%20Gas.md) · [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) →

Solidity tidak dijalankan langsung. Compiler (`solc`) mengubahnya jadi **bytecode**: deretan
byte, dan tiap byte adalah satu instruksi (**opcode**) untuk EVM. Contoh awal hampir semua
kontrak Solidity:

```
60 80   PUSH1 0x80
60 40   PUSH1 0x40
52      MSTORE
```

"Chain EVM" artinya node di chain itu punya interpreter yang memahami **daftar opcode yang
sama dengan arti yang sama** (`0x01` = ADD, `0x55` = SSTORE, dst.) plus aturan gas yang
(hampir) sama. Akibatnya file bytecode yang **persis sama** bisa di-deploy ke Ethereum, BSC,
Base, dll. tanpa compile ulang, dan perilakunya sama.

Bukti (2026-10-03): kontrak Multicall3 di `0xcA11bde05977b3631167028862bE2a173976CA11`
punya hash bytecode identik di Ethereum dan BSC:
`0xd5c15df687b16f2ff992fc8d767b4216323184a2bbc6ee2f9c398c318e770891`
(`cast code <addr> --rpc-url <rpc> | cast keccak`).

Solana tidak bisa menjalankan bytecode ini: SVM memakai format program lain (sBPF), jadi
kontraknya harus ditulis dan di-compile ulang untuk mesin itu.

Catatan: "sama" di sini berarti kompatibel, bukan identik 100%. Beberapa chain tertinggal
atau berbeda di opcode baru (misalnya `PUSH0` sempat belum didukung di beberapa L2), jadi
cek versi EVM target sebelum deploy.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.08 - Gas](EVM%2001.08%20-%20Gas.md) · [EVM 01.10 - Format Alamat 0x](EVM%2001.10%20-%20Format%20Alamat%200x.md) →
