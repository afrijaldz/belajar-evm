---
title: EVM 06 - Mekanisme Burn UNI di Fork
tags: [learning, evm, ethereum, uni, fork, tokenomics]
updated: 2026-10-24
---

# Mekanisme Burn UNI di Fork

Sesi 6 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), lanjutan dari [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md). Di sesi ini kamu
membedah mekanisme buyback-burn UNI: kontrak apa yang terlibat, siapa yang sebenarnya membeli
dan membakar UNI, lalu mencobanya sendiri di `anvil` fork Ethereum mainnet. Melengkapi
UNI Diperiksa dengan Saringan Lima Chain.

Hasil percobaan untuk materi ini ada di [EVM 06 - Hasil Praktik](Hasil%20Praktik/EVM%2006%20-%20Hasil%20Praktik.md).

## Burn dikerjakan bot, bukan protokol

Transaksi burn UNI dikirim oleh EOA bot ke **kontrak bot** miliknya sendiri, bukan oleh
protokol. Transaksinya besar: ratusan log dan jutaan gas, dengan calldata ribuan byte dan
selector yang tidak ada di database publik.

Alur di dalam satu transaksi atomik:

1. Flash loan WETH (misalnya dari Morpho Blue).
2. Tarik fee protokol dari puluhan pool Uniswap v2 (`Mint` LP fee) dan v3 (`CollectProtocol`)
   ke **TokenJar**.
3. Beli tepat sejumlah UNI sebesar `threshold` dari beberapa pool.
4. Panggil **Firepit** → UNI sebesar `threshold` ke `0xdead`, TokenJar menyerahkan isinya ke bot.
5. Jual semua token hasil jar ke WETH, lunasi flash loan.

## Kontrak

| Kontrak | Peran |
| --- | --- |
| **Firepit** | penjaga gerbang: tarik UNI → `0xdead`, perintahkan jar melepas |
| **TokenJar** | wadah fee; `releaser` = Firepit |

Parameter Firepit:

| Parameter | Arti |
| --- | --- |
| `RESOURCE` | token yang harus dibayar (UNI) |
| `RESOURCE_RECIPIENT` | penerima pembayaran (`0x…dEaD`) |
| `threshold` | jumlah UNI per pemanggilan — "harga tiket" |
| `nonce` | naik 1 setiap `release` berhasil |
| `MAX_RELEASE_LENGTH` | jumlah aset maksimal per `release` (20) |
| `owner` | Timelock governance UNI |

Alamat dan nilai parameter di mainnet ada di [EVM 06 - Hasil Praktik](Hasil%20Praktik/EVM%2006%20-%20Hasil%20Praktik.md).

### Seluruh logika: 4 baris

```solidity
function release(uint256 _nonce, Currency[] calldata assets, address recipient) external handleNonce(_nonce) {
    require(assets.length <= MAX_RELEASE_LENGTH, TooManyAssets());
    RESOURCE.safeTransferFrom(msg.sender, RESOURCE_RECIPIENT, threshold);
    TOKEN_JAR.release(assets, recipient);
    emit Released(_nonce, recipient, assets);
}
```

- **Siapa pun boleh memanggil.** Tidak ada `onlyOwner`.
- **Nonce wajib sama dengan nonce kontrak** → dua bot berebut, hanya satu menang.
- **Pemanggil memilih aset** yang diambil, maksimal 20.

## Kenapa TokenJar biasanya kosong

Nilai fee tidak menumpuk di jar, tapi **mengendap sebagai fee protokol di masing-masing pool**
sampai ditarik. Bot menarik fee **dan** membakar UNI dalam transaksi yang sama, jadi di antara
dua burn, saldo TokenJar praktis nol.

**Protokol tidak pernah membeli UNI.** Protokol cuma menetapkan harga tiket (`threshold`). Bot
yang menghitung kapan fee di pool lebih besar dari nilai tiket, menanggung risiko, dan bersaing.
Persaingan itu yang menjaga nilai burn ≈ nilai fee.

Akibatnya, memanggil `release` sendiri saat jar kosong = membakar UNI sebesar `threshold` tanpa
mendapat apa-apa. Bot butuh jutaan gas karena menarik fee dari puluhan pool dan swap dalam
transaksi yang sama; `release` sendiri cukup puluhan ribu gas.

## Firepit di L2

Selain Firepit di Ethereum mainnet, ada **Firepit di L2** dengan `threshold` lebih kecil (2.000
UNI). Setelah `release`, Firepit L2 mem-bridge UNI ke `0xdead` di L1 lewat `_afterRelease`.
Jadi tidak semua transfer UNI ke `0xdead` di L1 datang dari Firepit mainnet; sebagian datang dari
bridge (L1StandardBridge Base/Optimism, L1ERC20Gateway Arbitrum, dan gateway/NTT lain). Lihat
[EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md).

## Custom error di-encode seperti fungsi

Memanggil `release` dengan nonce salah menghasilkan revert `custom error 0x756688fe`. Nilai itu
sama dengan `cast sig 'InvalidNonce()'`. **Custom error di-encode seperti fungsi** (4 byte
keccak nama), lebih hemat dari `Error(string)`.

## Jebakan: token governance punya buku besar kedua

Slot mapping `balances` UNI = **4**. Layout lain: slot 0 `totalSupply`, slot 1 `minter`, slot 2
`mintingAllowedAfter`.

Memberi saldo UNI lewat `anvil_setStorageAt` membuat `balanceOf` benar, tapi transfer bisa revert
**`Uni::_moveVotes: vote amount underflows`**. Sebabnya:

- UNI punya buku besar kedua: **checkpoint suara** governance.
- Kalau akun itu sudah pernah self-delegate di mainnet, transfer akan mengurangi suaranya.
- Saldo ditulis tanpa menambah checkpoint → transfer mengurangi suara dari 0 → underflow.

Akun #0 `anvil` (`0xf39F…2266`) **sudah self-delegate di mainnet**, karena private key-nya publik
dan banyak orang memakainya. Jadi akun itu terkena masalah ini.

Ini contoh nyata peringatan [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md): menulis storage bisa terlihat berhasil
lalu gagal saat dipakai. Solusi: pakai akun baru yang belum pernah delegate.

## Aturan mint UNI

Hanya `minter` (Timelock) yang boleh mint, dengan dua batas:

- `mintCap` = 2% dari `totalSupply` per mint.
- `minimumTimeBetweenMints` = 31.536.000 detik (365 hari).

| Percobaan | Hasil |
| --- | --- |
| mint melebihi 2% | ❌ `Uni::mint: exceeded mint cap` |
| mint tepat 2% | ✅ |
| mint lagi sebelum 365 hari | ❌ `minting not allowed yet` |
| mint dari akun selain `minter` | ❌ `only the minter can mint` |

**Detail penting:** batas 2% dihitung dari `totalSupply`, yang **masih termasuk UNI di `0xdead`**.
Burn ke `0xdead` tidak mengurangi `totalSupply`, jadi terhadap supply efektif, mint 2% sebenarnya
sedikit lebih dari 2%.

## Jebakan tooling

- zsh **tidak memecah variabel tanpa kutip** (`set -- $t` gagal). Logika multi-kolom → Python.
- Output `cast` menyertakan anotasi (`31536000 [3.153e7]`) → `cut -d' ' -f1` sebelum aritmetika.
- Fork Ethereum di blok terbaru bisa jalan dengan full node (state ~25 menit). Pilih RPC yang
  stabil; pakai `--retries`, `--fork-retry-backoff`, dan `--timeout` di `anvil`.

## Coba sendiri

Jalankan `anvil` fork Ethereum mainnet, lalu:

1. Baca `threshold()` dan `nonce()` Firepit.
2. Panggil `release` dengan nonce salah, lalu cocokkan data revert dengan `cast sig 'InvalidNonce()'`.
3. Beri saldo UNI ke akun #0 `anvil` lewat storage, lalu coba transfer (harus revert karena
   checkpoint suara).
4. Ulangi dengan akun baru: beri saldo, `approve` Firepit, lalu `release`. Cek saldo `0xdead` dan
   apa yang kamu terima dari jar.
5. Impersonate Timelock sebagai `minter`, lalu coba mint di atas 2%, tepat 2%, dan mint kedua.

```bash
anvil --fork-url https://eth.drpc.org --retries 10 --fork-retry-backoff 2000 --timeout 60000
cast call <firepit> 'threshold()(uint256)' ; cast call <firepit> 'nonce()(uint256)'
cast rpc anvil_setStorageAt <uni> $(cast index address <akun> 4) $(cast to-uint256 4000000000000000000000)
cast send <uni> 'approve(address,uint256)' <firepit> 4000000000000000000000 --private-key <pk>
cast send <firepit> 'release(uint256,address[],address)' <nonce> "[<weth>,0x0000000000000000000000000000000000000000]" <akun> --private-key <pk>
cast call <uni> 'transferFrom(address,address,uint256)(bool)' <a> <b> <n> --from <spender> --trace
```
