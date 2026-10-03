---
title: EVM 02.01 - EOA dan Contract Account
tags: [learning, evm, account]
updated: 2026-10-03
---

# EOA dan Contract Account

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)

Di EVM hanya ada **dua jenis akun**. Bedanya ada pada **siapa yang mengendalikan** akun itu.

- **EOA (Externally Owned Account)**: akun yang dikendalikan oleh **private key** yang dipegang
  seseorang **di luar** chain (di wallet, HP, hardware wallet). "Externally owned" = pemiliknya
  ada di luar blockchain. Contoh: wallet MetaMask kamu.
- **Contract account**: akun yang dikendalikan oleh **kode (bytecode)** yang tersimpan di akun
  itu sendiri. **Tidak punya private key**, jadi tidak ada orang yang bisa "login" ke kontrak.
  Ia hanya bergerak kalau dipanggil, dan perilakunya persis sesuai kodenya. Contoh: token USDC,
  WETH, pool Uniswap.

Analogi: EOA = **orang** yang punya rekening dan tanda tangan. Contract account = **mesin
otomatis** (mesin minuman) yang punya laci uang sendiri: tidak bisa bergerak sendiri, tapi
kalau ada yang memasukkan koin dan menekan tombol, ia menjalankan aturan yang sudah tertanam,
dan aturan itu tidak bisa diubah siapa pun.

## Gambaran

```text
   Orang (private key)                          Tidak ada orang di balik kontrak
         │ tanda tangan                                     
         ▼                                                  
 ┌─────────────────────┐    transaksi     ┌─────────────────────┐   CALL   ┌─────────────────────┐
 │ EOA                 │ ───────────────► │ Contract A          │ ───────► │ Contract B          │
 │ balance: 5,7 ETH    │                  │ balance             │          │ balance             │
 │ nonce: 5966         │                  │ nonce: 1            │          │ nonce: 1            │
 │ code: (kosong)      │                  │ code: bytecode      │          │ code: bytecode      │
 │ storage: (kosong)   │                  │ storage: data       │          │ storage: data       │
 └─────────────────────┘                  └─────────────────────┘          └─────────────────────┘
   bisa MEMULAI transaksi                   hanya BEREAKSI saat dipanggil, lalu boleh memanggil kontrak lain
```

## Perbandingan

| | EOA | Contract account |
| --- | --- | --- |
| Dikendalikan oleh | private key (orang / program di luar chain) | kode di dalam akun |
| Punya private key? | ya | **tidak** |
| `code` | kosong `0x` (kecuali delegasi EIP-7702) | bytecode, tidak bisa diubah |
| `storage` | kosong | dipakai kontrak untuk datanya |
| Bisa **memulai** transaksi? | **ya, satu-satunya** | tidak; hanya bisa memanggil kontrak lain *di dalam* transaksi yang dimulai EOA |
| Cara lahir | otomatis: alamat dihitung dari public key, tidak perlu transaksi | dibuat lewat transaksi deploy (`CREATE`/`CREATE2`) |
| Alamat dari | `keccak256(public key)` 20 byte terakhir (lihat [EVM 01.10 - Format Alamat 0x](../EVM%2001%20-%20Detail/EVM%2001.10%20-%20Format%20Alamat%200x.md)) | `keccak256(rlp(deployer, nonce))` atau rumus `CREATE2` |
| `nonce` artinya | jumlah transaksi yang sudah dikirim | jumlah kontrak yang sudah dibuat, mulai dari 1 |
| Kalau kunci hilang | dana hilang selamanya | tidak ada kunci; aman/tidaknya tergantung kodenya |

Cara membedakan dari luar: **cek `code`**. `cast code <alamat>` → `0x` berarti EOA (atau
alamat yang belum pernah dipakai); ada bytecode berarti kontrak. Tidak ada field "tipe akun".

Akibat penting:

- **Setiap transaksi pasti dimulai EOA.** Kontrak yang "jalan otomatis" (misalnya burn mingguan
  CAKE) sebenarnya dipicu oleh bot EOA (lihat [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md), bagian BSC). Di dalam
  kontrak, `tx.origin` selalu EOA, `msg.sender` bisa EOA atau kontrak (lihat
  [EVM 01.05 - Environment](../EVM%2001%20-%20Detail/EVM%2001.05%20-%20Environment.md)).
- **Kontrak tidak bisa menyimpan rahasia atau menandatangani apa pun**, karena tidak punya kunci
  dan storage-nya bisa dibaca semua orang (lihat [EVM 01.06 - Storage](../EVM%2001%20-%20Detail/EVM%2001.06%20-%20Storage.md)).
- **Wallet "smart account"** (Safe, ERC-4337) adalah contract account yang memegang dana, tapi
  tetap butuh EOA (pemilik/bundler) untuk mengirim transaksinya.

## Garis yang mulai kabur: EIP-7702

Sejak upgrade Pectra (2025), EOA boleh **menitipkan code** berupa penunjuk 23 byte
`0xef0100 + alamat kontrak`. EOA itu tetap punya private key dan tetap bisa memulai transaksi,
tapi kalau dipanggil, ia menjalankan logika kontrak tujuan. Detail dan percobaannya di
[EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) (bagian EIP-7702).

## Bukti (2026-10-03)

Akun nyata di Ethereum mainnet:

| Akun | code | nonce | Jenis |
| --- | ---: | ---: | --- |
| `0xd8dA…6045` (wallet publik Vitalik Buterin) | **23 byte** `0xef01005a7f…6f6d` | 5.966 | EOA dengan delegasi EIP-7702 ke kontrak `0x5a7f…6f6d` (11.162 byte) |
| `0xA0b8…eB48` (USDC) | 2.186 byte | 1 | contract (proxy) |
| `0xC02a…6Cc2` (WETH) | 3.124 byte | 1 | contract |
| `0x…dEaD` | 0 | 0 | EOA tanpa pemilik yang diketahui |

Membuat masing-masing di anvil:

- **EOA**: `cast wallet new` membuat key + alamat **secara offline**, tanpa transaksi. Alamat
  baru `0x8830…cB91` langsung "ada" di chain dengan nonce 0, code `0x`, saldo 0.
- **Contract**: `cast compute-address 0xf39F…2266 --nonce 58` memprediksi `0xc351…1181`;
  deploy berikutnya dari akun itu mendarat **persis** di alamat tersebut, dengan nonce 1 dan
  code `0x00`. Alamat kontrak murni hasil rumus, jadi tidak ada private key yang menghasilkannya.

---

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)
