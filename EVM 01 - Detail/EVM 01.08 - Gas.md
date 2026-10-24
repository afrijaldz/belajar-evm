---
title: EVM 01.08 - Gas
tags: [learning, evm, gas]
updated: 2026-10-24
---

# Gas

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) · [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) →

Hasil percobaan untuk materi ini ada di [EVM 01.08 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.08%20-%20Hasil%20Praktik.md).

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

Tiga akibat yang perlu diingat:

- **Loop tanpa akhir pasti berhenti.** Loop `0x5b600056` (`JUMPDEST · PUSH1 0 · JUMP`) memakan
  12 gas per putaran (`JUMPDEST` 1 + `PUSH1` 3 + `JUMP` 8), jadi ia berhenti dengan `OutOfGas`
  begitu gas habis.
- **Gagal tetap bayar.** Transaksi yang `OutOfGas` berstatus 0 dan gas used = **seluruh gas
  limit**. Biayanya tetap gas limit × gas price.
- **Sukses hanya bayar yang terpakai.** Transaksi yang berhasil membayar gas used × gas price;
  sisa gas limit tidak dibayar. Contohnya loop 3 → 0 dari [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md):
  gas used 21.000 + 81 = 21.081.

## Coba sendiri

1. Deploy loop tanpa akhir `0x5b600056` di `anvil`, panggil dengan gas limit 21.100 (sisa 100
   untuk eksekusi), lalu lihat gas turun per langkah dengan `cast run <tx> -t`.
2. Panggil lagi dengan gas limit 50.000. Cek status dan gas used, lalu bandingkan penurunan
   saldo pengirim dengan gas used × gas price.
3. Panggil loop 3 → 0 dengan gas limit 50.000 dan bandingkan dengan langkah 2.

```bash
cast run <tx> -t                 # sisa gas per opcode
cast receipt <tx>                # status, gasUsed, effectiveGasPrice
cast balance <addr>              # saldo sebelum dan sesudah, dalam wei
```

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.07 - Program Counter](EVM%2001.07%20-%20Program%20Counter.md) · [EVM 01.09 - Bytecode yang Sama](EVM%2001.09%20-%20Bytecode%20yang%20Sama.md) →
