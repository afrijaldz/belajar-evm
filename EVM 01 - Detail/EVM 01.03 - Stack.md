---
title: EVM 01.03 - Stack
tags: [learning, evm, stack]
updated: 2026-10-03
---

# Stack

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.02 - Opcode](EVM%2001.02%20-%20Opcode.md) · [EVM 01.04 - Memory dan Calldata](EVM%2001.04%20-%20Memory%20dan%20Calldata.md) →

Stack adalah tumpukan kotak. Setiap kotak (disebut
**word**) lebarnya tetap **32 byte = 256 bit**, tidak bisa lebih kecil atau lebih besar.
Maksimal 1024 kotak; kalau lebih, eksekusi gagal (*stack overflow*). Opcode hanya bisa
mengambil dan menaruh kotak di **paling atas** (seperti tumpukan piring).

```text
         ┌──────────────────────── 32 byte (64 karakter hex) ────────────────────────┐
top  →   │ 0x0000000000000000000000000000000000000000000000000000000000000005        │  angka 5
         │ 0x000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266        │  alamat (20 byte, dipad 0 di kiri)
         │ 0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8        │  hash keccak256("hello")
         └───────────────────────────────────────────────────────────────────────────┘
          ... maks 1024 kotak
```

Akibatnya:

- **Semua nilai jadi 32 byte.** Angka kecil (5), alamat (20 byte), `bool`, semuanya diisi nol
  di kiri sampai 32 byte. Ini juga alasan argumen ABI selalu kelipatan 32 byte
  (lihat [EVM 04 - ABI](../EVM%2004%20-%20ABI.md)).
- **Kenapa 256 bit:** pas dengan ukuran hash `keccak256` (32 byte) dan angka kriptografi
  secp256k1 yang dipakai tanda tangan, jadi satu kotak muat satu hash atau satu kunci.
- **Angka terbesar = `type(uint256).max`** = 2²⁵⁶ − 1 = 115792089237316195423570985008687907853269984665640564039457584007913129639935
  (78 digit). Di level opcode, `ADD` yang melewati batas **memutar balik** ke 0 (modulo 2²⁵⁶),
  tanpa error. Solidity ≥0.8 menambahkan pengecekan sendiri supaya kasus ini revert
  (lihat [EVM 07 - Dasar Solidity](../EVM%2007%20-%20Dasar%20Solidity.md)).
- **Storage juga per 32 byte:** satu slot = key 32 byte → value 32 byte. Karena itu variabel
  kecil bisa "dipacking" ke satu slot untuk hemat gas.

Bukti (2026-10-03, anvil): bytecode `PUSH32 0xff…ff (max) · PUSH1 2 · ADD · PUSH1 0 · SSTORE ·
CALLER · PUSH1 1 · SSTORE` → slot 0 = `0x…01` (max + 2 memutar jadi 1, tanpa revert),
slot 1 = `0x000000000000000000000000f39fd6e5…92266` (alamat pemanggil dipad ke 32 byte).

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.02 - Opcode](EVM%2001.02%20-%20Opcode.md) · [EVM 01.04 - Memory dan Calldata](EVM%2001.04%20-%20Memory%20dan%20Calldata.md) →
