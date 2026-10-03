---
title: EVM 01.06 - Storage
tags: [learning, evm, storage]
updated: 2026-10-03
---

# Storage

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) · [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) →

Penyimpanan **permanen** milik setiap kontrak, bagian dari world state yang
disimpan semua node. Bentuknya tabel key → value: **2²⁵⁶ slot**, key 32 byte, value 32 byte.
Semua slot awalnya 0, jadi slot yang belum pernah ditulis tidak memakan tempat. Inilah tempat
"data aplikasi" hidup: saldo token, pemilik kontrak, konfigurasi.

```text
Kontrak UNI 0x1f98…F984 (mainnet, 2026-10-03)
┌─────────────────────────────────┬──────────────────────────────────────────────────────┐
│ key (slot)                      │ value (32 byte)                                      │
├─────────────────────────────────┼──────────────────────────────────────────────────────┤
│ 0                               │ 0x…033b2e3c9fd0803ce8000000 = 10²⁷ → totalSupply     │
│ 1                               │ 0x…1a9c8182c09f50c8318d769245bea52c32be35bc → minter │
│ 2                               │ 0x…65920080 = 2024-01-01 → mintingAllowedAfter       │
│ 3                               │ 0 (mapping allowances, isinya di slot lain)          │
│ 4                               │ 0 (mapping balances, isinya di slot lain)            │
│ keccak256(pad(0xdEaD) ‖ pad(4)) │ 112.515.581,08 UNI → balances[0xdEaD]                │
│   = 0x42c6…47dd                 │                                                      │
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

| Operasi | Gas | Terukur (anvil, termasuk PUSH/CALLDATALOAD) |
| --- | ---: | ---: |
| `SLOAD` slot pertama kali dalam tx (*cold*) | 2.100 | 2.103 |
| `SLOAD` slot yang sama lagi (*warm*) | 100 | 2.206 (cold + warm) |
| `SSTORE` 0 → bukan 0 (slot baru) | 22.100 | 22.109 |
| `SSTORE` bukan 0 → bukan 0 lain | 5.000 | 5.009 |
| `SSTORE` nilai sama | 2.200 | 2.209 |
| `SSTORE` bukan 0 → 0 (hapus) | 5.000, refund 4.800 di akhir tx | 5.009 |

Bandingkan: `MSTORE` ke memory 3 gas, `SSTORE` slot baru 22.100 gas (~7.000x).

Bukti (2026-10-03): UNI dibaca dengan `cast storage` ke `ethereum-rpc.publicnode.com`;
slot 0 sama dengan `totalSupply()` = 10²⁷, nilai di
`cast index address 0x…dEaD 4` sama dengan `balanceOf(0xdEaD)` = 112515581083689919081184389,
sedangkan key yang sama dengan slot 3 dan 5 bernilai 0 (jadi `balances` ada di slot 4).
Tabel gas dari kontrak kecil di anvil, diukur dengan `cast run -vvvvv`.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) · [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) →
