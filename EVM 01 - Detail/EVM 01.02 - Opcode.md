---
title: EVM 01.02 - Opcode
tags: [learning, evm, opcode]
updated: 2026-10-03
---

# Opcode

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.01 - Mesin EVM](EVM%2001.01%20-%20Mesin%20EVM.md) · [EVM 01.03 - Stack](EVM%2001.03%20-%20Stack.md) →

Singkatan *operation code*: **satu perintah dasar** untuk EVM, ditulis sebagai
**satu byte** (nilai `0x00`–`0xff`). Bytecode kontrak hanyalah deretan opcode. Setiap opcode
punya nomor (byte), nama singkat (*mnemonic*) untuk dibaca manusia, aturan ambil/taruh di
stack, dan harga gas. Analoginya: instruksi mesin CPU (`ADD`, `MOV`, `JMP` di x86), tapi untuk
mesin virtual.

```text
byte     60      80      60      40      52       ← yang disimpan di chain
         │       │       │       │       │
nama     PUSH1   (data)  PUSH1   (data)  MSTORE   ← mnemonic, hasil disassemble
arti     taruh 0x80      taruh 0x40      simpan 0x80 ke memory offset 0x40
```

Dari 256 nilai byte, hanya ~150 yang terdefinisi. Byte yang tidak terdefinisi (dan `0xfe`
`INVALID`) membuat eksekusi gagal. Opcode baru ditambahkan lewat hard fork (mis. `PUSH0`
`0x5f` di Shanghai 2023, `TLOAD`/`TSTORE`/`MCOPY` di Cancun 2024), dan inilah sumber perbedaan
kecil antar chain EVM.

| Kelompok | Contoh opcode (byte) | Gas |
| --- | --- | ---: |
| Hitung | `ADD 01`, `MUL 02`, `SUB 03`, `DIV 04`, `EXP 0a` | 3–5 (EXP lebih) |
| Bandingkan & bit | `LT 10`, `GT 11`, `EQ 14`, `ISZERO 15`, `AND 16`, `SHR 1c` | 3 |
| Hash | `KECCAK256 20` | 30 + per word |
| Environment | `ADDRESS 30`, `CALLER 33`, `CALLVALUE 34`, `CALLDATALOAD 35`, `NUMBER 43`, `CHAINID 46` | 2–3 |
| Stack | `POP 50`, `PUSH0 5f`, `PUSH1 60` … `PUSH32 7f`, `DUP1 80` …, `SWAP1 90` … | 2–3 |
| Memory | `MLOAD 51`, `MSTORE 52`, `MSIZE 59` | 3 + perluasan |
| Storage | `SLOAD 54`, `SSTORE 55`, `TLOAD 5c`, `TSTORE 5d` | 100–22.100 |
| Alur | `JUMP 56`, `JUMPI 57`, `PC 58`, `JUMPDEST 5b` | 1–10 |
| Event | `LOG0 a0` … `LOG4 a4` | 375 + per topic/byte |
| Panggil kontrak | `CREATE f0`, `CALL f1`, `DELEGATECALL f4`, `CREATE2 f5`, `STATICCALL fa` | 100+ |
| Selesai | `STOP 00`, `RETURN f3`, `REVERT fd`, `INVALID fe` | 0 |

Solidity → opcode: compiler menerjemahkan setiap baris jadi banyak opcode. `a + b` jadi beberapa
`DUP`/`ADD`/pengecekan overflow; memilih fungsi jadi `CALLDATALOAD`, `SHR`, `EQ`, `JUMPI`.
Daftar lengkap + gas + simulator: https://www.evm.codes/

Bukti (2026-10-03): `cast disassemble $(cast code 0xcA11…CA11)` (Multicall3, Ethereum)
→ 3.808 byte = 2.160 opcode, 78 jenis berbeda. Awalnya:

```text
00: PUSH1 0x80       ┐ siapkan memory (pola awal hampir semua kontrak Solidity)
02: PUSH1 0x40       │
04: MSTORE           ┘
05: PUSH1 0x04       ┐ calldata < 4 byte? → lompat ke fallback
07: CALLDATASIZE     │
08: LT               │
09: PUSH2 0x00f3     │
0c: JUMPI            ┘
0d: PUSH1 0x00       ┐ ambil selector: 32 byte pertama calldata,
0f: CALLDATALOAD     │ geser kanan 0xe0 = 224 bit → sisa 4 byte
10: PUSH1 0xe0       │
12: SHR              ┘
13: DUP1
14: PUSH4 0x4d2301cc   ← selector getEthBalance(address) (dicek dengan cast sig)
```

Yang paling sering: `PUSH` 552, `DUP` 396, `SWAP` 190, `JUMPDEST` 151, `POP` 143: sebagian besar
kerja EVM adalah mengatur stack. Byte tak terdefinisi `0x0c` dijalankan di anvil → status 0,
gas used 50.000 = seluruh gas limit.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.01 - Mesin EVM](EVM%2001.01%20-%20Mesin%20EVM.md) · [EVM 01.03 - Stack](EVM%2001.03%20-%20Stack.md) →
