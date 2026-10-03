---
title: EVM 01.10 - Format Alamat 0x
tags: [learning, evm, address]
updated: 2026-10-03
---

# Format Alamat 0x

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) · [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md) →

Dua hal yang terpisah:

1. **`0x` cuma penanda heksadesimal**, konvensi dari bahasa C. Bukan bagian dari alamatnya.
   Alamat EVM sebenarnya adalah **20 byte** = 40 karakter hex.
2. **Kenapa 20 byte:** alamat EOA diturunkan dari kunci:
   private key → public key (secp256k1, 64 byte) → `keccak256(public key)` (32 byte) →
   ambil **20 byte terakhir**.

Bukti dengan key default anvil #0 (2026-10-03):

```
public key   = 0x8318535b…0d2aa5 (64 byte)
keccak256    = 0xc1ffd3cfee2d9e5cd67643f8 f39fd6e51aad88f6f4ce6ab8827279cfffb92266
alamat       =                            0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

Huruf besar-kecil campuran di alamat adalah **checksum EIP-55**: kapitalisasi dihitung dari
hash alamat itu sendiri, jadi salah ketik satu karakter bisa dideteksi wallet.

Karena semua chain EVM memakai rumus ini, **satu private key = alamat yang sama di semua
chain EVM**. Solana berbeda: alamatnya public key ed25519 32 byte yang ditulis dalam base58
(tanpa `0x`), dan Bitcoin pakai format lain lagi (base58/bech32, mis. `bc1…`).

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) · [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md) →
