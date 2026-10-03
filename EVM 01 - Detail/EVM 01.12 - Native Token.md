---
title: EVM 01.12 - Native Token
tags: [learning, evm, native-token]
updated: 2026-10-03
---

# Native Token

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md)

**Native token** adalah koin yang **dibangun langsung ke dalam protokol chain**, bukan dibuat
oleh smart contract. ETH di Ethereum, BNB di BSC, POL di Polygon, AVAX di Avalanche C-Chain.
Setiap chain EVM punya tepat satu, dan dialah satu-satunya alat bayar gas di chain itu.

## Disimpan di mana

Ingat model akun: setiap akun di world state punya empat field, `nonce`, `balance`, `code`,
`storage` (lihat [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)). Saldo native token **adalah field `balance`
itu sendiri**. Token lain (ERC-20) hanyalah angka di dalam **storage** sebuah kontrak.

```text
World state
├─ akun 0xf39F… (EOA)
│    balance: 10.000 ETH          ← native token: field bawaan setiap akun
│    nonce, code (kosong), storage (kosong)
│
└─ akun 0xC02a… (kontrak WETH)
     balance: 2.027.136 ETH       ← ETH asli yang dititipkan ke kontrak
     code: bytecode WETH
     storage:
       totalSupply → 2.027.136 WETH
       balanceOf[0xf39F…] → …     ← token ERC-20: cuma angka di storage kontrak
```

## Native token vs token ERC-20

| | Native token (ETH, BNB di BSC) | Token ERC-20 (USDT, UNI, WETH) |
| --- | --- | --- |
| Dibuat oleh | protokol chain | smart contract (siapa pun bisa deploy) |
| Disimpan di | field `balance` akun | storage kontrak token (`mapping balances`) |
| Alamat kontrak | **tidak ada** | ada (mis. UNI `0x1f98…F984`) |
| Cek saldo | `eth_getBalance` (`cast balance`) | `eth_call` ke `balanceOf(address)` |
| Kirim | isi field `value` di transaksi | panggil `transfer(to, amount)` |
| Event `Transfer` | **tidak ada** log | ada, dari kontrak |
| `approve` / `transferFrom` | tidak ada | ada |
| Bayar gas | **ya, satu-satunya** | tidak bisa |
| Supply baru dari | aturan protokol (reward validator), dikurangi burn (EIP-1559, BEP-95) | fungsi `mint` di kontrak |
| Satuan terkecil | wei (1 ETH = 10¹⁸ wei) | `decimals()` milik kontrak |

Akibat praktisnya:

- **Indexer tidak bisa melacak transfer native lewat `eth_getLogs`**, karena tidak ada event.
  Transfer native di dalam kontrak (internal transfer) hanya terlihat lewat `trace_*`
  (lihat [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md)).
- **Kontrak DeFi butuh versi ERC-20 dari native token** supaya bisa `approve`/`transferFrom`:
  inilah **WETH** (Wrapped ETH) atau WBNB. Setor 1 ETH → dapat 1 WETH; tukar balik kapan saja.
- **Native token tidak punya `totalSupply()`**. Supply ETH harus dihitung dari luar
  (riwayat reward dan burn), tidak bisa dibaca dengan satu `eth_call`.

## Nama sama, jenis beda

Native atau bukan itu soal **chain**, bukan soal nama koin:

- **BNB di BSC** = native token (gas, `eth_getBalance`).
- **BNB di Ethereum** = token ERC-20 biasa di `0xB8c7…DD52`, sisa era 2017 sebelum BNB pindah
  ke chain sendiri. Tidak bisa dipakai bayar gas di Ethereum.
- **"ETH" di BSC** = token BEP-20 (Binance-Peg), bukan native (lihat [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md)).

## Bukti (2026-10-03)

| Cek | Hasil |
| --- | --- |
| Transfer 1 ETH ke `0x…bEEF` di anvil | status 1, gas **21.000** (`0x5208`, hanya biaya dasar), **0 log**; saldo 0 → 1 ETH; code penerima `0x` (EOA biasa) |
| BNB di Ethereum (`0xB8c7…DD52`) | ada code 5.355 byte, `symbol()` = `"BNB"`, `totalSupply()` = 392.908,13 BNB → kontrak ERC-20 |
| BNB di BSC, `eth_getBalance(0xdEaD)` | 16.498.950,81 BNB, naik dari 16.498.949,12 beberapa jam sebelumnya (burn BNB jalan terus) |
| WETH mainnet `0xC02a…6Cc2` | saldo ETH native kontrak = `totalSupply()` WETH = **2.027.136,551081447271723759**, sama sampai wei terakhir |

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.11 - RPC eth](EVM%2001.11%20-%20RPC%20eth.md)
