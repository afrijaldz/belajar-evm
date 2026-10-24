---
title: EVM 04 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 04 - Hasil Praktik

Hasil percobaan untuk materi [EVM 04 - ABI](../EVM%2004%20-%20ABI.md). Dijalankan 2026-09-24 dengan `cast` (Foundry
1.7.1), `anvil` lokal, dan satu transaksi nyata di BSC. Project:
`~/Documents/riset/evm/account-model` (kontrak `AbiLab` dengan event indexed string dan event
anonymous).

## Selector dan event di anvil

- `cast 4byte 0x42966c68` mengembalikan dua nama: `burn(uint256)` dan
  `collate_propagate_storage(bytes16)`.
- Event `Note` di `AbiLab` dengan tag `"burn"`: `topics[2]` = `keccak256("burn")`, `data` memakai
  offset `0x40` dan `0x80`.
- Event anonymous `Ping(99)` di `AbiLab` menghasilkan `topics: []`.

## Studi kasus: transaksi mint-burn CAKE

Transaksi `0xb7a7e2989258cde2b25942484f051d3168e65b9db0d0d11f750f211d780abd11` (BSC,
2026-09-21 06:51 UTC) dari CAKE Diperiksa Ulang 2026-09-24. Calldata 1.476 byte.

### Lapisan calldata

```
EOA bot 0xd099… (nonce 6.725)
 └─ Safe 0xeCc9….execTransaction (selector 0x6a761202)
      tanda tangan 195 byte = 3 × 65 byte → 3 signer
      operation = 1 (delegatecall)
     └─ MultiSend 0x40A2….multiSend(bytes) (0x8d80ff0a)
          format packed: op (1) | to (20) | value (32) | panjang (32) | data
         ├─ Timelock 0xa1f4….executeTransaction(address,uint256,string,bytes,uint256) (0x0825f38f)
         │    target MasterChefV2, signature "burnCake(bool)", data true, eta 2026-09-21 01:00 UTC
         └─ Timelock 0xa1f4….queueTransaction(...) (0x3a66f901)
              target MasterChefV2, signature "burnCake(bool)", data true, eta 2026-09-28 01:00 UTC
```

- **Siklusnya mingguan.** Selisih eta = 604.800 detik = 7 hari. Tiap minggu multisig
  menjalankan perintah yang diantrekan minggu lalu dan mengantrekan perintah minggu depan.
- **Timelock memaksa jeda 7 hari**, jadi perintah minggu depan sudah bisa dibaca on-chain hari
  ini lewat `queueTransaction`.

### 17 log

| Log | Emitter | Event | Arti |
| ---: | --- | --- | --- |
| 0–1 | Safe + `0x53a1…78f1` | `SafeMultiSigTransaction`, `SafeTxData` | catatan eksekusi multisig |
| 2–8 | MasterChefV2 | 7× `UpdatePool` | 7 pool dummy diperbarui |
| 9 | CAKE | `Transfer` 0x0 → Safe burn `0xceba…`, **530.328** | mint 10% dev share |
| 10 | CAKE | `Transfer` 0x0 → SyrupBar `0x009c…`, **5.303.280** | mint 90% |
| 11 | CAKE | `Transfer` SyrupBar → MasterChefV2, 5.303.280 | diteruskan |
| 12 | MasterChefV1 `0x73fe…` | `Deposit(user=MCv2, pid=526, amount=0)` | amount 0 = **panen**, bukan setor |
| 13 | CAKE | `Transfer` MasterChefV2 → Safe burn, **54.049.639** | `burnCake` |
| 14–15 | Timelock | `ExecuteTransaction`, `QueueTransaction` | eksekusi + antre |
| 16 | Safe | `ExecutionSuccess` | selesai |

Terverifikasi: `0x009c…` = `"SyrupBar Token"`; `MasterChefV1.cakePerBlock()` = **40 CAKE**.

**Belum cocok saat itu:** di transaksi ini MCv2 cuma menerima 5,3 jt tapi mengirim 54,0 jt ke Safe
burn, padahal saldo MCv2 di titik-titik sampel selalu kecil (~0,37 jt). Dugaan: sisanya dipanen
oleh transaksi lain sepanjang minggu. Pertanyaan ini direncanakan dijawab di Sesi 16, dan
terjawab di [EVM 05 - Hasil Praktik](EVM%2005%20-%20Hasil%20Praktik.md) (saldo MCv2 menumpuk sepanjang minggu).

## Koreksi ke catatan sebelumnya

Catatan [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) semula menyebut bot `0xd099…` memicu mint-burn **harian**. Dari studi kasus ini
terlihat yang benar **mingguan** — sudah diperbaiki di [EVM 02 - Hasil Praktik](EVM%2002%20-%20Hasil%20Praktik.md) dan di
CAKE Diperiksa Ulang 2026-09-24.

## Jebakan tooling yang ditemui

Dua jebakan di bagian "Jebakan" materi (argumen negatif ke `cast calldata`, `pkill -f` membunuh
shell sendiri) ditemui di sesi ini.
