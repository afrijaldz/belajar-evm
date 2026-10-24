---
title: EVM 01.10 - Format Alamat 0x
tags: [learning, evm, address]
updated: 2026-10-24
---

# Format Alamat 0x

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) · [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.10 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.10%20-%20Hasil%20Praktik.md).

Dua hal yang terpisah:

1. **`0x` cuma penanda heksadesimal**, konvensi dari bahasa C. Bukan bagian dari alamatnya.
   Alamat EVM sebenarnya adalah **20 byte** = 40 karakter hex.
2. **Kenapa 20 byte:** alamat EOA diturunkan dari kunci:
   private key → public key (secp256k1, 64 byte) → `keccak256(public key)` (32 byte) →
   ambil **20 byte terakhir**.

Contoh dengan key default anvil #0. Alamatnya adalah 20 byte terakhir dari hash public key:

```
public key   = 0x8318535b…0d2aa5 (64 byte)
keccak256    = 0xc1ffd3cfee2d9e5cd67643f8 f39fd6e51aad88f6f4ce6ab8827279cfffb92266
alamat       =                            0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

Huruf besar-kecil campuran di alamat adalah **checksum EIP-55**: kapitalisasi dihitung dari
hash alamat itu sendiri, jadi salah ketik satu karakter bisa dideteksi wallet.

Karena semua chain EVM memakai rumus ini, **satu private key = alamat yang sama di semua
chain EVM**. Chain non-EVM seperti Bitcoin memakai format lain (base58/bech32, mis. `bc1…`).

## Coba sendiri

1. Ambil private key akun #0 dari output `anvil`.
2. Turunkan public key-nya, lalu hash dengan `keccak256`.
3. Bandingkan 20 byte terakhir hash dengan alamat akun #0.

```bash
cast wallet public-key --raw-private-key <pk>      # public key 64 byte
cast keccak <public-key>                           # hash 32 byte
cast wallet address --private-key <pk>             # alamat (sudah dengan checksum EIP-55)
cast to-check-sum-address <alamat-huruf-kecil>     # hitung checksum EIP-55
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) · [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md) →
