---
title: EVM 01.06 - Storage
tags: [learning, evm, storage]
updated: 2026-10-24
---

# Storage

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) · [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.06 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.06%20-%20Hasil%20Praktik.md).

Penyimpanan **permanen** milik setiap kontrak, bagian dari world state yang
disimpan semua node. Bentuknya tabel key → value: **2²⁵⁶ slot**, key 32 byte, value 32 byte.
Semua slot awalnya 0, jadi slot yang belum pernah ditulis tidak memakan tempat. Inilah tempat
"data aplikasi" hidup: saldo token, pemilik kontrak, konfigurasi.

Contoh: layout storage kontrak token UNI (`0x1f98…F984`, Ethereum mainnet).

```text
┌─────────────────────────────────┬──────────────────────────────────────────────────────┐
│ key (slot)                      │ value (32 byte)                                      │
├─────────────────────────────────┼──────────────────────────────────────────────────────┤
│ 0                               │ totalSupply                                          │
│ 1                               │ minter (alamat, dipad ke 32 byte)                    │
│ 2                               │ mintingAllowedAfter (timestamp)                      │
│ 3                               │ 0 (mapping allowances, isinya di slot lain)          │
│ 4                               │ 0 (mapping balances, isinya di slot lain)            │
│ keccak256(pad(0xdEaD) ‖ pad(4)) │ balances[0xdEaD]                                     │
├─────────────────────────────────┴──────────────────────────────────────────────────────┤
│ ... 2²⁵⁶ slot, semua 0 kecuali yang pernah ditulis                                     │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

Cara Solidity menaruh variabel (*storage layout*):

- Variabel state diberi nomor slot berurutan sesuai urutan deklarasi (0, 1, 2, …). Variabel
  kecil yang berurutan bisa dipacking ke satu slot. `constant`/`immutable` tidak memakai
  storage (ditanam di bytecode).
- `mapping` hanya "memesan" satu slot (tetap 0). Isinya disimpan di
  `keccak256(key ‖ nomor slot mapping)`, jadi tersebar acak di ruang 2²⁵⁶ dan tidak bertabrakan.
  Hitung dengan `cast index address <key> <slot>`.

Sifat penting:

- **Permanen**: bertahan antar transaksi, sampai ditimpa. Bandingkan stack dan memory yang
  hilang setelah call.
- **Hanya kontrak itu yang bisa menulis** storage-nya sendiri (`SSTORE`). Kontrak lain tidak
  bisa membaca langsung; harus memanggil fungsinya.
- **Tapi semua orang bisa membaca dari luar** lewat `eth_getStorageAt` (`cast storage`).
  Variabel `private` di Solidity tetap terbaca; `private` hanya mencegah kontrak lain
  memanggilnya. Jangan simpan rahasia di storage.
- **Paling mahal**, karena setiap node harus menyimpannya selamanya.

| Operasi | Gas |
| --- | ---: |
| `SLOAD` slot pertama kali dalam tx (*cold*) | 2.100 |
| `SLOAD` slot yang sama lagi (*warm*) | 100 |
| `SSTORE` 0 → bukan 0 (slot baru) | 22.100 |
| `SSTORE` bukan 0 → bukan 0 lain | 5.000 |
| `SSTORE` nilai sama | 2.200 |
| `SSTORE` bukan 0 → 0 (hapus) | 5.000, refund 4.800 di akhir tx |

Bandingkan: `MSTORE` ke memory 3 gas, `SSTORE` slot baru 22.100 gas (~7.000x).

## Coba sendiri

1. Baca slot 0–2 kontrak UNI dengan `cast storage`, lalu bandingkan slot 0 dengan
   `totalSupply()`.
2. Cari slot mapping `balances`: hitung `cast index address 0x…dEaD <slot>` untuk slot 3, 4,
   dan 5, baca masing-masing, lalu bandingkan dengan `balanceOf(0xdEaD)`. Hanya slot yang benar
   yang memberi nilai bukan 0 dan sama dengan `balanceOf`.
3. Ukur gas `SLOAD`/`SSTORE` dengan kontrak kecil di `anvil` dan `cast run -vvvvv`. Angkanya
   sedikit di atas tabel karena ikut menghitung opcode pendukung (`PUSH`, `CALLDATALOAD`).

```bash
cast storage <kontrak> <slot> --rpc-url <rpc>          # baca slot mentah
cast index address <key> <slot-mapping>                # slot isi mapping
cast call <kontrak> "totalSupply()(uint256)" --rpc-url <rpc>
cast call <kontrak> "balanceOf(address)(uint256)" <addr> --rpc-url <rpc>
cast run <tx> -vvvvv                                   # gas per call
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) · [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) →
