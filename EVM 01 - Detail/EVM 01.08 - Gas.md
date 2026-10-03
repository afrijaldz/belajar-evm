---
title: EVM 01.08 - Gas
tags: [learning, evm, gas]
updated: 2026-10-03
---

# Gas

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) · [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) →

Satuan **jumlah kerja** yang dilakukan EVM. Setiap opcode punya harga tetap
dalam gas: `ADD` 3, `JUMP` 8, `SLOAD` 2.100, `SSTORE` slot baru 22.100, dan setiap transaksi
bayar dasar 21.000. Gas **bukan** mata uang; gas diubah ke uang lewat *gas price*.

Kenapa perlu gas:

1. **Menghentikan loop tanpa akhir.** EVM tidak bisa tahu sebelumnya apakah sebuah program
   akan berhenti (*halting problem*). Dengan gas, setiap langkah memakan "bensin", jadi program
   apa pun pasti berhenti saat bensinnya habis.
2. **Mencegah spam/DoS.** Menyuruh ribuan node bekerja tidak gratis.
3. **Membayar validator** yang menjalankan dan menyimpan hasilnya.
4. **Membatasi ukuran blok.** Setiap blok punya *block gas limit*, jadi kerja per blok terbatas.

Di dalam mesin, gas adalah penghitung yang turun setiap langkah, seperti tangki bensin:

```text
gas limit (diisi pengirim)  50.000  ███████████████████████████████████
  − 21.000 biaya dasar tx           ████████████████████▌
  − tiap opcode (ADD 3, ...)        ████████████████████▊
  = gas used                        selesai → sisa dikembalikan, tidak dibayar
                                    habis   → OutOfGas: semua perubahan batal, gas tetap dibayar
biaya (wei) = gas used × gas price (wei per gas)
```

Tiga angka yang perlu dibedakan:

| Angka | Siapa yang menentukan | Arti |
| --- | --- | --- |
| Gas limit | pengirim tx | batas maksimal kerja yang mau dibayar |
| Gas used | hasil eksekusi | kerja yang benar-benar terpakai |
| Gas price | pasar (base fee + tip, detail di [EVM 03 - Transaksi dan Gas](../EVM%2003%20-%20Transaksi%20dan%20Gas.md)) | harga per unit gas dalam wei |

Opcode `GAS` (`gasleft()` di Solidity) membaca sisa penghitung ini.

Bukti (2026-10-03, anvil):

- **Loop tanpa akhir** `0x5b600056` (`JUMPDEST · PUSH1 0 · JUMP`), gas limit 21.100 (sisa 100
  untuk eksekusi): `cast run -t` menunjukkan gas turun 100 → 99 → 96 → 88 → 87 → … (12 gas per
  putaran), berhenti setelah 27 langkah dengan `OutOfGas`.
- **Gagal tetap bayar:** loop tanpa akhir dengan limit 50.000 → status 0, gas used **50.000**
  (seluruh limit), saldo pengirim turun 44.296.250.000 wei = 50.000 × 885.925 wei, persis.
- **Sukses bayar yang terpakai saja:** loop 3 → 0 dengan limit 50.000 → status 1, gas used
  **21.081** (21.000 + 81), saldo turun 16.349.453.874 wei = 21.081 × 775.554, persis; sisa
  28.919 gas tidak dibayar.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) · [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) →
