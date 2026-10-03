---
title: EVM 06 - Mekanisme Burn UNI di Fork
tags: [learning, evm, ethereum, uni, fork, tokenomics]
updated: 2026-09-24
---

# Mekanisme Burn UNI di Fork

Catatan dari Sesi 6 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Rekonstruksi buyback-burn UNI dari transaksi
nyata, lalu dijalankan sendiri di `anvil` fork Ethereum mainnet (blok ~26.046.123, RPC
`eth.drpc.org`). Lanjutan dari [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md); melengkapi
UNI Diperiksa dengan Saringan Lima Chain.

## Transaksi burn asli: dikerjakan bot, bukan protokol

Burn terbaru `0x0bdd8b780b66ae68c52eebb7f4660eaddb63b85265549dae86c41d692c422aaa`
(2026-09-23 20:31 UTC):

| | |
| --- | --- |
| Pengirim | EOA `0xFeeFee6e90b7Bc64f6ADBa1dDb6d5A13c2D169d3` |
| Tujuan | kontrak bot `0x9b715efB4EbDB14862d079A841bA4983FFd69236` (19.767 byte) |
| Calldata | 8.228 byte, selector `0x7e874012` (tidak ada di database publik) |
| **Log / gas** | **256 log, 4.236.883 gas** |

Alur di dalam satu transaksi atomik:

1. Flash loan WETH dari Morpho Blue `0xbbbbbbbbbb9cc5e90e3b3af64bdaf62c37eeffcb`.
2. Tarik fee protokol dari puluhan pool Uniswap v2 (`Mint` LP fee) dan v3 (`CollectProtocol`)
   ke **TokenJar**.
3. Beli tepat **4.000 UNI** dari beberapa pool.
4. Panggil **Firepit** → 4.000 UNI ke `0xdead`, TokenJar menyerahkan semua isinya ke bot.
5. Jual semua token hasil jar ke WETH, lunasi flash loan.

## Kontrak

| Kontrak | Alamat | Peran |
| --- | --- | --- |
| **Firepit** | `0x0d5cd355e2abeb8fb1552f56c965b867346d6721` | penjaga gerbang: tarik UNI → `0xdead`, perintahkan jar melepas |
| **TokenJar** | `0xf38521f130fccf29db1961597bc5d2b60f995f85` | wadah fee; `releaser` = Firepit |

Konfigurasi Firepit (2026-09-24):

| Parameter | Nilai |
| --- | --- |
| `RESOURCE` | UNI |
| `RESOURCE_RECIPIENT` | `0x…dEaD` |
| `threshold` | **4.000 UNI** (~$36.960 di $9,24) |
| `nonce` | 1.361 (transaksi di atas = 1.359) |
| `MAX_RELEASE_LENGTH` | 20 aset |
| `owner` | Timelock governance UNI `0x1a9C…35BC` |

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

## Temuan kunci: TokenJar kosong saat istirahat

Di fork, saldo TokenJar: ETH, WETH, USDC, USDT, WBTC, UNI, DAI — **semua 0**.

Nilainya tidak menumpuk di jar, tapi **mengendap sebagai fee protokol di masing-masing pool**
sampai ditarik. Bot menarik fee **dan** membakar UNI dalam transaksi yang sama.

**Protokol tidak pernah membeli UNI.** Protokol cuma menetapkan harga tiket (4.000 UNI). Bot yang
menghitung kapan fee di pool > 4.000 UNI, menanggung risiko, dan bersaing. Persaingan itu yang
menjaga nilai burn ≈ nilai fee — penjelasan kenapa burn on-chain cocok ±3–6% dengan holder
revenue DefiLlama di riset UNI.

## Percobaan di fork

### 1. Nonce salah

`release(1360, …)` → `custom error 0x756688fe` = `cast sig 'InvalidNonce()'`. **Custom error di-encode
seperti fungsi** (4 byte keccak nama), lebih hemat dari `Error(string)`.

### 2. Invariant rusak karena menulis storage

Slot mapping `balances` UNI = **4** (dicocokkan dengan saldo `0xdead`). Layout lain:
slot 0 `totalSupply`, slot 1 `minter`, slot 2 `mintingAllowedAfter`.

Akun `anvil` #0 `0xf39F…2266` diberi 4.000 UNI lewat `anvil_setStorageAt` → `balanceOf` bilang
4.000, tapi transfer revert **`Uni::_moveVotes: vote amount underflows`**. Sebabnya:

- UNI punya buku besar kedua: **checkpoint suara** governance.
- Akun #0 **sudah self-delegate di mainnet** (private key-nya publik, banyak orang memakainya).
- Saldo ditulis tanpa menambah checkpoint → transfer mengurangi suara dari 0 → underflow.

Contoh nyata peringatan [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md): menulis storage bisa terlihat berhasil lalu
gagal saat dipakai. Solusi: akun baru yang belum pernah delegate.

### 3. Burn sendiri dari akun baru

| | |
| --- | --- |
| status | sukses, 64.879 gas, 3 log |
| `0xdead` | **+4.000 UNI** |
| Yang diterima | **0 WETH, 0 ETH** (jar kosong) |
| Firepit nonce | 1.361 → 1.362 |
| Hasil | rugi total ~$36.960 |

Bot asli butuh 4,24 jt gas karena menarik fee dari puluhan pool dan swap dalam transaksi yang sama.

### 4. Aturan mint UNI (impersonate Timelock sebagai `minter`)

`mintCap` = 2%, `minimumTimeBetweenMints` = 31.536.000 detik (365 hari).

| Percobaan | Hasil |
| --- | --- |
| mint 2% + 1 wei | ❌ `Uni::mint: exceeded mint cap` |
| **mint tepat 2% (20 jt)** | ✅ `totalSupply` 1,00 M → **1,02 M** |
| mint lagi 1 UNI | ❌ `minting not allowed yet` — terbuka lagi **2027-09-24** |
| mint dari akun biasa | ❌ `only the minter can mint` |

**Detail baru:** batas 2% dihitung dari `totalSupply` = 1 miliar, yang **masih termasuk 112 jt di
`0xdead`**. Terhadap supply efektif (~887,8 jt), 20 jt = **2,25%**, bukan 2%.

## Angka yang bergerak hari ini

> ⚠️ **Koreksi Sesi 22 ([EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md)):** +68.000 UNI di bawah ini **bukan** dari Firepit mainnet
> (nonce cuma naik 1.359 → 1.361). Sejak 2026-09-23 06:00 UTC: 51 transfer, 112.000 UNI ke `0xdead`, hanya
> 4.000 dari Firepit mainnet. Sisanya **Firepit di L2 (threshold 2.000)** yang mem-bridge UNI ke `0xdead` di L1
> lewat `_afterRelease` — datang dari L1StandardBridge Base/Optimism, L1ERC20Gateway Arbitrum, dan
> gateway/NTT lain.

Saldo `0xdead` UNI: 112.091.581 (riset, pagi) → **112.159.581** (fork, sore) = **+68.000 UNI dalam
beberapa jam**.

## Jebakan tooling

- zsh **tidak memecah variabel tanpa kutip** (`set -- $t` gagal). Logika multi-kolom → Python.
- Output `cast` menyertakan anotasi (`31536000 [3.153e7]`) → `cut -d' ' -f1` sebelum aritmetika.
- Fork Ethereum di blok terbaru jalan dengan full node (state ~25 menit). publicnode sempat
  timeout saat `anvil` start; `eth.drpc.org` stabil.

## Perintah

```bash
anvil --fork-url https://eth.drpc.org --retries 10 --fork-retry-backoff 2000 --timeout 60000
cast call <firepit> 'threshold()(uint256)' ; cast call <firepit> 'nonce()(uint256)'
cast rpc anvil_setStorageAt <uni> $(cast index address <akun> 4) $(cast to-uint256 4000000000000000000000)
cast send <uni> 'approve(address,uint256)' <firepit> 4000000000000000000000 --private-key <pk>
cast send <firepit> 'release(uint256,address[],address)' <nonce> "[<weth>,0x0000000000000000000000000000000000000000]" <akun> --private-key <pk>
cast call <uni> 'transferFrom(address,address,uint256)(bool)' <a> <b> <n> --from <spender> --trace
```
