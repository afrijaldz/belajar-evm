---
title: EVM 01.04 - Memory dan Calldata
tags: [learning, evm, memory]
updated: 2026-10-24
---

# Memory dan Calldata

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.03 - Stack](EVM%2001.03%20-%20Stack.md) · [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.04 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.04%20-%20Hasil%20Praktik.md).

Keduanya deretan byte (bukan tumpukan kotak seperti stack),
dibaca per alamat byte (*offset*). Bedanya ada di asal data dan siapa yang boleh menulis.

```text
CALLDATA  (input dari pemanggil, READ-ONLY)
offset:   0         4                                   36                                  68
          │a9059cbb │000…000000000000000000000000dEaD   │000…00000000000000000000000003e8   │
          │selector │arg 1: to (32 byte)                │arg 2: amount = 1000 (32 byte)     │
          transfer(address,uint256)

MEMORY  (kertas coret-coretan milik satu call, BISA DITULIS)
offset:   0x00                0x20                0x40               ...
          │ 0x…2a (MSTORE)    │ 0 (belum dipakai) │ 0                │  mulai kosong, membesar sesuai kebutuhan
```

| | Calldata | Memory |
| --- | --- | --- |
| Isi | input transaksi/call: selector 4 byte + argumen ABI | data sementara selama eksekusi |
| Siapa yang mengisi | pemanggil (wallet atau kontrak lain) | kode kontrak itu sendiri |
| Bisa ditulis? | **tidak** | ya |
| Opcode | `CALLDATALOAD` (ambil 32 byte), `CALLDATASIZE`, `CALLDATACOPY` (salin ke memory) | `MLOAD`, `MSTORE` (32 byte), `MSTORE8` (1 byte), `MSIZE` |
| Umur | selama call; isinya tetap tercatat di transaksi | **hilang** begitu call selesai; tiap call mulai dari nol |
| Biaya | dibayar per byte di transaksi (lihat [EVM 03 - Transaksi dan Gas](../EVM%2003%20-%20Transaksi%20dan%20Gas.md)); membaca murah (3 gas) | 3 gas per akses + biaya **membesarkan** memory yang naik kuadratik |
| Di Solidity | parameter `calldata` (hanya untuk fungsi `external`) | parameter/variabel `memory`, string/array sementara, data `return` |

Memory dipakai karena stack hanya muat kotak 32 byte: data yang panjang (string, array,
data `return`, data `LOG`, argumen untuk memanggil kontrak lain) harus disusun dulu di memory.
`RETURN` dan `LOG` membaca dari memory, bukan dari stack.

Contoh dengan calldata `transfer(0xdEaD, 1000)`:

- `CALLDATASIZE` = 68 (= 4 + 32 + 32).
- `CALLDATALOAD(0)` = `0xa9059cbb000…`: selector + 28 byte awal argumen pertama, karena
  `CALLDATALOAD` selalu mengambil 32 byte.
- **Memory tidak dibawa ke call berikutnya.** `MLOAD(0)` di awal call selalu 0, walaupun call
  sebelumnya sudah menulis ke offset itu.
- `MSIZE` dihitung per 32 byte: setelah satu `MSTORE` di offset 0, `MSIZE` = 32.

### Biaya membesarkan memory

Biaya memory dihitung dari ukuran memory dalam word (32 byte): **3 × word + word² / 512**.
Karena ada suku kuadrat, biaya naik jauh lebih cepat daripada ukurannya. Satu `MSTORE` di
offset ~1 MB (32.768 word) sudah memakan sekitar 2,2 juta gas, jadi memory besar cepat sekali
jadi tidak terjangkau.

## Coba sendiri

1. Buat kontrak kecil di `anvil` yang membaca `CALLDATASIZE` dan `CALLDATALOAD(0)`, menyimpan
   `MLOAD(0)`, lalu menulis 42 ke memory offset 0 dan menyimpan `MSIZE`.
2. Panggil dua kali dengan calldata `transfer(0xdEaD, 1000)` dan bandingkan hasilnya.
3. Ganti offset `MSTORE` (0, 1.024, 32.768, 1.048.576) dan bandingkan gas eksekusinya.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.03 - Stack](EVM%2001.03%20-%20Stack.md) · [EVM 01.05 - Environment](EVM%2001.05%20-%20Environment.md) →
