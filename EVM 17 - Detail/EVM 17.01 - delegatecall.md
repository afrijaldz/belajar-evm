---
title: EVM 17.01 - delegatecall
tags: [learning, evm, solidity, delegatecall, proxy, security]
updated: 2026-10-03
---

# delegatecall

Bagian dari [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md) · terkait: [EVM 07.02 - Fallback](../EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md)

**`delegatecall` adalah cara menjalankan kode kontrak lain seolah-olah kode itu milik
kontrak sendiri.** Kodenya dipinjam dari kontrak B, tetapi semua yang lain tetap milik
kontrak A: storage, saldo ETH, `address(this)`, `msg.sender`, dan `msg.value`.

Bandingkan dengan `call` biasa:

| | `call` (A → B) | `delegatecall` (A → B) |
| --- | --- | --- |
| Kode yang dijalankan | kode B | kode B |
| Storage yang dibaca/ditulis | **storage B** | **storage A** |
| `address(this)` | B | **A** |
| `msg.sender` di dalam kode B | A | **pemanggil asli A** (misalnya EOA) |
| `msg.value` | nilai yang dikirim A ke B | **nilai tx asli ke A** |
| ETH berpindah? | bisa (`call{value: x}`) | tidak pernah; tidak ada parameter value |

## Analogi: resep masakan

- **`call`** = kamu memesan makanan ke restoran B. Restoran B memasak di dapurnya sendiri,
  memakai bahan dari kulkasnya sendiri. Dapurmu tidak berubah.
- **`delegatecall`** = kamu meminjam **buku resep** restoran B, lalu memasak di **dapurmu
  sendiri** dengan bahan dari kulkasmu sendiri. Restoran B tidak tahu apa-apa dan dapurnya
  tidak berubah.

Resep itu menulis "ambil bahan dari rak nomor 0". Rak nomor 0 di dapurmu mungkin berisi hal
yang berbeda dari rak nomor 0 di restoran B. Itulah sumber bug tabrakan storage (lihat
[EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md#bug-1-tabrakan-storage)).

```
           call                                 delegatecall
EOA ─► A ─────────► B                  EOA ─► A ───────────────► B
                    │                         │                  │
              kode B jalan                    │◄── kode B dipinjam
              storage B                       │
              msg.sender = A            kode B jalan DI DALAM A
                                        storage A
                                        msg.sender = EOA
```

## Bukti test Foundry

File: `~/Documents/riset/evm/proxy/src/Delegatecall.sol` dan `test/Delegatecall.t.sol`.
`Probe.record(n)` menyimpan `number`, `msg.sender`, `msg.value`, dan `address(this)`.
`Caller` punya layout storage yang sama dan memanggil `record` dengan dua cara.

```bash
cd ~/Documents/riset/evm/proxy && forge test --match-path test/Delegatecall.t.sol -vv
```

Hasil: **5 passed, 0 failed**.

Alice memanggil `Caller` dengan 1 ETH dan `n = 7`:

| Hasil | `viaCall` | `viaDelegatecall` |
| --- | --- | --- |
| Storage yang berubah | `Probe` (`number` = 7) | **`Caller`** (`number` = 7) |
| Storage yang tetap 0 | `Caller` | `Probe` |
| `sender` yang tercatat | `Caller` | **alice** |
| `value` yang tercatat | 1 ETH | 1 ETH (`msg.value` tx asli) |
| `self` (`address(this)`) | `Probe` | **`Caller`** |
| ETH 1 ETH berakhir di | `Probe` | `Caller` |

Test lain:

| Test | Hasil |
| --- | --- |
| `delegatecall` ke alamat **tanpa kode** | `ok = true`, return data 0 byte, tidak ada yang berubah |
| `delegatecall` ke fungsi yang revert | `ok = false`, return data = `Error(string)` "probe failed"; revert tidak diteruskan otomatis |
| `UnsafeWallet.execute(target, data)` yang memakai `delegatecall`, dipanggil attacker dengan `TakeOver.pwn()` | `owner` wallet berubah menjadi **attacker**; storage `TakeOver` sendiri tetap kosong |

## Opcode

| Opcode | Byte | Argumen stack |
| --- | --- | --- |
| `CALL` | `0xf1` | gas, addr, **value**, argsOffset, argsSize, retOffset, retSize (7) |
| `DELEGATECALL` | `0xf4` | gas, addr, argsOffset, argsSize, retOffset, retSize (6, **tanpa value**) |
| `STATICCALL` | `0xfa` | sama dengan `DELEGATECALL`, tetapi tidak boleh mengubah state |
| `CALLCODE` | `0xf2` | versi lama; storage milik A, tetapi `msg.sender` di kode B = kontrak A, bukan pemanggil asli. Diganti `DELEGATECALL` (EIP-7) |

Disassembly `Caller` berisi 2 `DELEGATECALL` (`viaDelegatecall` dan `rawDelegatecall`) dan
1 `CALL` (`viaCall`).

## Kegunaan

1. **Proxy dan upgrade.** Proxy menyimpan state dan alamat; kontrak logika hanya menyediakan
   kode. Upgrade = ganti alamat logika, state tetap (lihat [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md) dan
   contoh USDC di [EVM 07.02 - Fallback](../EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md)).
2. **Library.** Fungsi `public`/`external` di `library` Solidity dipanggil dengan
   `delegatecall`, sehingga kode library bisa memakai storage kontrak pemanggil.
3. **Minimal proxy / clone (EIP-1167).** Ribuan kontrak kecil (sekitar 45 byte) berbagi satu
   kontrak logika. Setiap clone punya storage sendiri.
4. **EIP-7702** bekerja dengan ide yang sama: EOA menjalankan kode kontrak lain di atas
   storage EOA sendiri (lihat [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)).

## Bahaya

- **`delegatecall` ke kode yang tidak dipercaya = menyerahkan kontrak.** Kode itu bisa menulis
  slot mana pun, termasuk `owner`, dan bisa memindahkan semua ETH. Test `UnsafeWallet`
  membuktikan ini dalam satu transaksi.
- **Alamat tanpa kode tetap "sukses".** Kalau alamat logika salah atau kontraknya hilang,
  `delegatecall` mengembalikan `true` tanpa menjalankan apa pun. Proxy yang baik memeriksa
  `target.code.length > 0` sebelum dipakai.
- **Layout storage harus sama.** Kode B menulis berdasarkan nomor slot, bukan nama variabel.
  Variabel yang urutannya berbeda di A akan tertimpa.
- **Insiden nyata:** dompet multisig Parity (2017) memakai satu kontrak library bersama lewat
  `delegatecall`. Seseorang meng-`initialize` library itu, menjadi owner, lalu memanggil
  `selfdestruct`. Semua wallet yang bergantung pada library itu kehilangan kodenya dan dananya
  terkunci (sekitar 513.000 ETH). Angka ini dari laporan publik dan tidak dicek ulang di sesi ini.

## Hal yang sering membingungkan

- **`delegatecall` tidak membuat kontrak B "bekerja untuk" A.** B tidak tahu apa-apa dan
  storage B tidak berubah. Yang dipakai hanya bytecode B.
- **Kontrak yang dipanggil lewat `delegatecall` tidak menerima ETH.** ETH tetap di kontrak
  pemanggil, tetapi kode B tetap bisa membaca `msg.value` dari tx asli.
- **Constructor B tidak pernah berjalan di A.** Constructor hanya berjalan saat B di-deploy,
  di storage B. Karena itu proxy memakai fungsi `initialize`.

---

Bagian dari [EVM 17 - Proxy dan Upgrade](../EVM%2017%20-%20Proxy%20dan%20Upgrade.md) · terkait: [EVM 07.02 - Fallback](../EVM%2007%20-%20Detail/EVM%2007.02%20-%20Fallback.md)
