---
title: EVM 02.01 - EOA dan Contract Account
tags: [learning, evm, account]
updated: 2026-10-24
---

# EOA dan Contract Account

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) · [EVM 02.02 - Nonce](EVM%2002.02%20-%20Nonce.md) →

Hasil percobaan untuk materi ini ada di [EVM 02.01 - Hasil Praktik](../Hasil%20Praktik/EVM%2002.01%20-%20Hasil%20Praktik.md).

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
 │ balance: 2 ETH      │                  │ balance             │          │ balance             │
 │ nonce: 42           │                  │ nonce: 1            │          │ nonce: 1            │
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
  CAKE) sebenarnya dipicu oleh bot EOA (lihat
  [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md#kontrak-tidak-bisa-memulai-transaksi)). Di dalam
  kontrak, `tx.origin` selalu EOA, `msg.sender` bisa EOA atau kontrak (lihat
  [EVM 01.05 - Environment](../EVM%2001%20-%20Detail/EVM%2001.05%20-%20Environment.md)).
- **Kontrak tidak bisa menyimpan rahasia atau menandatangani apa pun**, karena tidak punya kunci
  dan storage-nya bisa dibaca semua orang (lihat [EVM 01.06 - Storage](../EVM%2001%20-%20Detail/EVM%2001.06%20-%20Storage.md)).
- **Wallet "smart account"** (Safe, ERC-4337) adalah contract account yang memegang dana, tapi
  tetap butuh EOA (pemilik/bundler) untuk mengirim transaksinya.

## Garis yang mulai kabur: EIP-7702

Sejak upgrade Pectra (2025), EOA boleh **menitipkan code** berupa penunjuk 23 byte
`0xef0100 + alamat kontrak`. EOA itu tetap punya private key dan tetap bisa memulai transaksi,
tapi kalau dipanggil, ia menjalankan logika kontrak tujuan. Detailnya di
[EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) (bagian "Pengecualian modern: EIP-7702").

Delegasi ini terlihat lewat `cast code`: EOA yang mendelegasikan code-nya punya code tepat
**23 byte** yang diawali `0xef0100`, tapi nonce-nya tetap menghitung transaksi yang ia kirim.
Wallet EOA yang sudah lama dipakai bisa saja punya code seperti ini.

## Cara lahir masing-masing

- **EOA** lahir **offline**. `cast wallet new` membuat key + alamat tanpa transaksi apa pun.
  Alamat baru itu langsung "ada" di chain dengan nonce 0, code `0x`, saldo 0.
- **Contract** lahir lewat transaksi deploy. Alamatnya bisa dihitung lebih dulu dengan
  `cast compute-address <deployer> --nonce <n>`, dan deploy berikutnya dari akun itu mendarat
  **persis** di alamat tersebut, dengan nonce 1. Alamat kontrak murni hasil rumus, jadi tidak
  ada private key yang menghasilkannya.

## Coba sendiri

1. Cek `code` dan `nonce` beberapa akun nyata di Ethereum mainnet: wallet pribadi, token
   (USDC `0xA0b8…eB48`, WETH `0xC02a…6Cc2`), dan `0x…dEaD`.
2. Di `anvil`, buat EOA baru dengan `cast wallet new`, lalu cek nonce, code, dan saldonya.
3. Prediksi alamat kontrak dari akun #0, deploy kontrak, lalu cek alamat, nonce, dan code-nya.

```bash
cast code <alamat> --rpc-url <rpc>        # 0x = EOA; ada bytecode = kontrak
cast codesize <alamat> --rpc-url <rpc>
cast nonce <alamat> --rpc-url <rpc>
cast wallet new                           # key + alamat baru, offline
cast compute-address <deployer> --nonce $(cast nonce <deployer>)
```

---

Bagian dari [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md) · [EVM 02.02 - Nonce](EVM%2002.02%20-%20Nonce.md) →
