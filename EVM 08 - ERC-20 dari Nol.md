---
title: EVM 08 - ERC-20 dari Nol
tags: [learning, evm, solidity, erc20, foundry]
updated: 2026-09-24
---

# ERC-20 dari Nol

Catatan dari Sesi 8 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Token ditulis sendiri tanpa library, diuji
dengan Foundry (14 test, 2 di antaranya fuzz × 256 run), lalu di-deploy ke `anvil`. Lanjutan dari
[EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md).

Project: `~/Documents/riset/evm/solidity-basics` — `src/MyToken.sol`, `test/MyToken.t.sol`.

## Isinya cuma dua mapping

```solidity
mapping(address => uint256) public balanceOf;
mapping(address => mapping(address => uint256)) public allowance;
```

Ditambah `totalSupply`, `name`, `symbol`, `decimals`, fungsi `transfer` / `approve` /
`transferFrom`, dan event `Transfer` / `Approval`. Sekitar 60 baris. Bytecode **3.091 byte** (CAKE
7.285 byte — sisanya delegasi/checkpoint suara dan `mint` owner).

## Selector dan event identik dengan token nyata

| | MyToken | Dipakai di riset |
| --- | --- | --- |
| `balanceOf(address)` | `0x70a08231` | ✓ `eth_call` CAKE, UNI, ETHFI |
| `totalSupply()` | `0x18160ddd` | ✓ |
| `transfer(address,uint256)` | `0xa9059cbb` | ✓ |
| `Transfer(address,address,uint256)` | `0xddf252ad…` | ✓ filter `eth_getLogs` burn |

`balanceOf` dan `allowance` di MyToken **bukan fungsi** — mapping `public` otomatis dapat getter
dengan selector yang sama. Script riset (eth_call selector mentah, `cast logs` topic Transfer)
jalan di token ini **tanpa perubahan**. Itulah gunanya standar.

## Keputusan desain

| Keputusan | Alasan |
| --- | --- |
| Mint di constructor = `emit Transfer(address(0), …)` | konvensi yang dipakai explorer dan indexer. Di Sesi 4, mint CAKE dikenali dari Transfer `0x0 → …` |
| Transfer ke `address(0)` ditolak | EIP-20 tidak melarang, tapi token hilang tanpa mengurangi `totalSupply` |
| Transfer nilai 0 diizinkan | EIP-20: *MUST be treated as normal transfers* |
| Allowance `type(uint256).max` tidak dikurangi | "unlimited approve" — hemat satu tulis storage |
| Custom error (`InsufficientBalance`, …) | lebih murah + membawa data (lihat [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md)) |

## Test

| Kelompok | Isi |
| --- | --- |
| Metadata | name, symbol, decimals, supply awal |
| Event | mint = Transfer dari 0x0; Transfer dan Approval dengan `vm.expectEmit` |
| Jalur gagal | saldo kurang, allowance kurang, penerima `address(0)` |
| Tepi | transfer ke diri sendiri, transfer 0, allowance unlimited |
| **Fuzz** | total saldo tetap = supply untuk `to`/`amount` acak; tidak bisa belanja melebihi allowance |

### Bug klasik yang ditangkap test

Menyimpan saldo penerima **sebelum** saldo pengirim dikurangi:

```solidity
uint256 toBalance = balanceOf[to];          // dibaca dulu
balanceOf[from] = fromBalance - value;
balanceOf[to] = toBalance + value;          // kalau from == to, pengurangan tertimpa
```

Transfer 100 ke diri sendiri → saldo **1.000.000 → 1.000.100**. Token tercetak dari udara,
`totalSupply` tidak berubah.

**Yang menangkap: test manual `test_TransferToSelfKeepsBalance`, bukan fuzz.** Fuzz test memakai
`vm.assume(to != alice)` sehingga tidak pernah mencoba transfer ke diri sendiri. **Setiap
pengecualian di fuzz = area yang tidak diuji.**

## Gas nyata (anvil, transaksi terpisah)

| Transaksi | Gas |
| --- | ---: |
| transfer ke holder **baru** | 52.349 |
| transfer ke holder **lama** | 35.249 |
| **Selisih** | **17.100** = SSTORE nol→bukan nol vs ubah nilai ([EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)) |
| approve (slot baru) | 46.710 |

Angka gas **di dalam** satu test Foundry menyesatkan: semua langkah berjalan di satu transaksi,
slot yang sudah disentuh jadi "warm". Contoh: "holder lama" di test = 5.272 gas, di anvil 35.249.
Untuk angka gas nyata, ukur dengan transaksi terpisah.

`transferFrom` dengan allowance terbatas vs unlimited (dalam test, relatif): 33.284 vs 27.981.

## Latihan: `burn`

`test/ExerciseBurn.t.sol` — tambahkan `burn(uint256 value)` ke `src/MyToken.sol`. Aturan:

- saldo pemanggil turun `value`
- **`totalSupply` turun `value`** — ini yang **tidak** dilakukan burn ke `0xdead` (UNI, CAKE)
- `emit Transfer(caller, address(0), value)`
- burn melebihi saldo → `InsufficientBalance`

```bash
cd ~/Documents/riset/evm/solidity-basics && forge test --match-contract ExerciseBurnTest -vv
```

Test memakai low-level call supaya file tetap ter-compile sebelum `burn` ada. Sudah diverifikasi
bisa diselesaikan. Status: ☐ belum dikerjakan

## Perintah

```bash
forge inspect MyToken methodIdentifiers      # selector semua fungsi
forge inspect MyToken events                 # topic semua event
forge create src/MyToken.sol:MyToken --private-key <pk> --broadcast --constructor-args "My Token" "MYT" 1000000000000000000000000
cast logs --from-block 0 --address <token> 'Transfer(address indexed,address indexed,uint256)'
```
