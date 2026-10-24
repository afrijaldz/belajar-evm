---
title: EVM 08 - ERC-20 dari Nol
tags: [learning, evm, solidity, erc20, foundry]
updated: 2026-10-24
---

# ERC-20 dari Nol

Sesi 8 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), lanjutan dari [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md). Di sesi ini kamu
menulis token ERC-20 sendiri tanpa library, mengujinya dengan Foundry (termasuk fuzz test), lalu
men-deploy-nya ke `anvil`.

Hasil percobaan untuk materi ini ada di [EVM 08 - Hasil Praktik](Hasil%20Praktik/EVM%2008%20-%20Hasil%20Praktik.md).

## Isinya cuma dua mapping

```solidity
mapping(address => uint256) public balanceOf;
mapping(address => mapping(address => uint256)) public allowance;
```

Ditambah `totalSupply`, `name`, `symbol`, `decimals`, fungsi `transfer` / `approve` /
`transferFrom`, dan event `Transfer` / `Approval`. Sekitar 60 baris. Token nyata biasanya lebih
besar karena fitur tambahan, misalnya delegasi dan checkpoint suara governance, atau fungsi
`mint` milik owner.

## Selector dan event identik dengan token nyata

| | Selector / topic |
| --- | --- |
| `balanceOf(address)` | `0x70a08231` |
| `totalSupply()` | `0x18160ddd` |
| `transfer(address,uint256)` | `0xa9059cbb` |
| `Transfer(address,address,uint256)` | `0xddf252ad…` |

`balanceOf` dan `allowance` di token ini **bukan fungsi** — mapping `public` otomatis dapat getter
dengan selector yang sama. Karena selector dan topic event-nya sama dengan token nyata, script
yang membaca token ERC-20 lewat selector mentah (`eth_call`) atau topic `Transfer` (`cast logs`)
jalan di token buatanmu **tanpa perubahan**. Itulah gunanya standar.

## Keputusan desain

| Keputusan | Alasan |
| --- | --- |
| Mint di constructor = `emit Transfer(address(0), …)` | konvensi yang dipakai explorer dan indexer untuk mengenali mint (lihat contoh mint CAKE di [EVM 04 - ABI](EVM%2004%20-%20ABI.md)) |
| Transfer ke `address(0)` ditolak | EIP-20 tidak melarang, tapi token hilang tanpa mengurangi `totalSupply` |
| Transfer nilai 0 diizinkan | EIP-20: *MUST be treated as normal transfers* |
| Allowance `type(uint256).max` tidak dikurangi | "unlimited approve" — hemat satu tulis storage |
| Custom error (`InsufficientBalance`, …) | lebih murah + membawa data (lihat [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md)) |

## Apa yang perlu diuji

| Kelompok | Isi |
| --- | --- |
| Metadata | name, symbol, decimals, supply awal |
| Event | mint = Transfer dari 0x0; Transfer dan Approval dengan `vm.expectEmit` |
| Jalur gagal | saldo kurang, allowance kurang, penerima `address(0)` |
| Tepi | transfer ke diri sendiri, transfer 0, allowance unlimited |
| **Fuzz** | total saldo tetap = supply untuk `to`/`amount` acak; tidak bisa belanja melebihi allowance |

### Bug klasik: transfer ke diri sendiri

Menyimpan saldo penerima **sebelum** saldo pengirim dikurangi:

```solidity
uint256 toBalance = balanceOf[to];          // dibaca dulu
balanceOf[from] = fromBalance - value;
balanceOf[to] = toBalance + value;          // kalau from == to, pengurangan tertimpa
```

Kalau `from == to`, pengurangan tertimpa, jadi saldo **bertambah** sebesar `value`. Token tercetak
dari udara, `totalSupply` tidak berubah.

**Jebakan:** fuzz test yang memakai `vm.assume(to != alice)` tidak pernah mencoba transfer ke diri
sendiri, jadi bug ini lolos dari fuzz. Yang menangkapnya test manual seperti
`test_TransferToSelfKeepsBalance`. **Setiap pengecualian di fuzz = area yang tidak diuji.**

## Gas: ukur dengan transaksi terpisah

- Transfer ke holder **baru** lebih mahal dari transfer ke holder **lama**. Selisihnya **17.100** =
  SSTORE nol→bukan nol vs ubah nilai ([EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)).
- `approve` pertama kali juga membuat slot baru.
- `transferFrom` dengan allowance terbatas lebih mahal dari allowance unlimited, karena allowance
  terbatas harus ditulis ulang.

**Jebakan:** angka gas **di dalam** satu test Foundry menyesatkan. Semua langkah berjalan di satu
transaksi, jadi slot yang sudah disentuh menjadi "warm" dan jauh lebih murah. Untuk angka gas
nyata, ukur dengan transaksi terpisah (misalnya di `anvil`).

## Latihan: `burn`

`test/ExerciseBurn.t.sol` — tambahkan `burn(uint256 value)` ke `src/MyToken.sol`. Aturan:

- saldo pemanggil turun `value`
- **`totalSupply` turun `value`** — ini yang **tidak** dilakukan burn ke `0xdead` (UNI, CAKE)
- `emit Transfer(caller, address(0), value)`
- burn melebihi saldo → `InsufficientBalance`

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-contract ExerciseBurnTest -vv
```

Test memakai low-level call supaya file tetap ter-compile sebelum `burn` ada. Status: ☐ belum
dikerjakan

## Bacaan lanjutan

- B20 Token Standard Base (folder `Learning/Base/B20/`): ERC-20 versi Base untuk saham tokenisasi. Selector sama dengan MyToken, ditambah role, pause, policy allowlist/blocklist, multiplier untuk dividen/split, dan factory precompile

## Coba sendiri

1. Tulis `MyToken` dengan dua mapping di atas, constructor yang me-mint supply awal, dan custom
   error.
2. Tulis test per kelompok di tabel "Apa yang perlu diuji", termasuk test manual transfer ke diri
   sendiri.
3. Cek selector dan topic dengan `forge inspect`, bandingkan dengan tabel selector di atas.
4. Deploy ke `anvil`, kirim transfer ke holder baru dan holder lama sebagai transaksi terpisah,
   lalu bandingkan `gasUsed`.
5. Baca semua event `Transfer` dengan `cast logs`.

```bash
forge inspect MyToken methodIdentifiers      # selector semua fungsi
forge inspect MyToken events                 # topic semua event
forge create src/MyToken.sol:MyToken --private-key <pk> --broadcast --constructor-args "My Token" "MYT" 1000000000000000000000000
cast logs --from-block 0 --address <token> 'Transfer(address indexed,address indexed,uint256)'
```
