---
title: EVM 09 - Bahaya Approval ERC-20
tags: [learning, evm, solidity, erc20, security]
updated: 2026-10-24
---

# Bahaya Approval ERC-20

Sesi 9 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu belajar kenapa `approve` adalah salah satu titik
paling berbahaya di ERC-20: empat skenario serangan (race condition, unlimited approve, phishing
`permit`, revoke) dan perilaku token nyata yang tidak patuh standar seperti USDT. Lanjutan dari
[EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md).

Hasil percobaan untuk materi ini ada di [EVM 09 - Hasil Praktik](Hasil%20Praktik/EVM%2009%20-%20Hasil%20Praktik.md).

## Dasar: approve menyerahkan kendali

`approve(spender, n)` = "`spender` boleh memindahkan sampai `n` token saya, kapan saja, ke mana
saja, tanpa tanya saya lagi". `transferFrom` tidak butuh tanda tangan pemilik — cukup allowance.

## Skenario 1: race condition saat menurunkan allowance

Alice mau menurunkan jatah Bob dari 100 ke 50. Bob melihat transaksi Alice di mempool:

1. Bob front-run: `transferFrom` 100 (allowance lama)
2. `approve(bob, 50)` Alice masuk — cuma **menimpa**, tidak tahu 100 sudah dipakai
3. Bob `transferFrom` 50 lagi

**Bob dapat 150**, padahal Alice bermaksud maksimal 100.

## Skenario 2: unlimited approve ke kontrak jahat

Alice approve `type(uint256).max` ke sebuah router hanya untuk deposit 10 token. Berbulan-bulan
kemudian dia menerima 500 token lagi, tanpa pernah menyentuh router itu lagi. Kalau router punya
pintu belakang (misalnya fungsi `sweep` milik admin), admin bisa menarik **seluruh 590 token**,
termasuk yang datang **setelah** approve.

- **Approval tidak pernah kedaluwarsa** dan berlaku untuk saldo masa depan.
- Kasus yang sama terjadi kalau router jujur tapi kunci admin-nya dicuri, atau ada bug.
- **Approve pas (10)** → setelah deposit allowance 0 → `sweep` revert. Kerusakan terbatas.

## Skenario 3: phishing tanda tangan `permit` (EIP-2612)

`permit` = approve lewat **tanda tangan off-chain**. Siapa pun yang memegang tanda tangan bisa
mengirimkannya.

Alice menandatangani "pesan login" yang sebenarnya permit unlimited untuk attacker. Akibatnya:

| | |
| --- | --- |
| Saldo Alice | bisa dikuras sampai **0** |
| Nonce transaksi Alice | **tidak berubah** — Alice tidak pernah mengirim transaksi |
| Gas | dibayar attacker |

Korban tidak melihat "Approve" di wallet, tidak bayar gas. Yang muncul hanya permintaan **"Sign
typed data"** berisi `spender`, `value`, `deadline` — field itu yang wajib dibaca sebelum tanda
tangan.

Dua proteksi bawaan permit:

| Serangan | Kenapa gagal |
| --- | --- |
| Tanda tangan dipakai dua kali | `nonces[owner]` sudah naik setelah pemakaian pertama, jadi digest-nya tidak cocok lagi |
| Tanda tangan dari satu chain dipakai di chain lain | `DOMAIN_SEPARATOR` mengikat chain ID + alamat kontrak |

Anatomi permit (EIP-712):

```
digest = keccak256("\x19\x01" ‖ DOMAIN_SEPARATOR ‖ keccak256(PERMIT_TYPEHASH, owner, spender, value, nonce, deadline))
signer = ecrecover(digest, v, r, s)   → harus == owner
```

### Penjelasan detail

1. [EVM 09.01 - Permit2](EVM%2009%20-%20Detail/EVM%2009.01%20-%20Permit2.md) — kontrak Uniswap yang memberi `permit` ke semua ERC-20 (termasuk USDT/WETH); mode SignatureTransfer vs AllowanceTransfer, contoh vault deposit, kenapa phishing Permit2 lebih berbahaya

## Skenario 4: revoke

`approve(spender, 0)` → `sweep` gagal, saldo aman. **Revoke melindungi masa depan, bukan masa
lalu** — token yang sudah ditarik tidak kembali.

## Token nyata: USDT (Ethereum, `0xdAC1…1ec7`)

USDT punya dua perilaku yang berbeda dari token patuh standar:

- **USDT memaksa aturan "nol dulu".** Mengubah allowance dari angka bukan nol langsung ke angka
  bukan nol lain (mis. 100 → 50) gagal, dan allowance tetap di angka lama. Harus `approve(0)` dulu,
  baru `approve(50)`. Ini mitigasi race condition, tapi **tidak sepenuhnya mencegah**: Bob tetap
  bisa menghabiskan 100 sebelum `approve(0)` masuk. Aturan ini hanya memberi kesempatan mengecek di
  antara dua langkah.
- **USDT tidak mengembalikan `bool`**, melanggar EIP-20. Return value `approve()` kosong (`0x`),
  sedangkan token patuh standar mengembalikan `true` (`…0001`). Kontrak dengan
  `require(token.approve(...))` akan revert dengan USDT karena tidak ada yang bisa di-decode.
  Inilah alasan `SafeERC20` (OpenZeppelin) ada. USDT dibuat sebelum EIP-20 final dan tidak bisa
  diubah.

## Kebiasaan aman

| Kebiasaan | Kenapa |
| --- | --- |
| Approve jumlah pas, bukan unlimited | membatasi kerusakan skenario 2 |
| Wallet hold terpisah, tidak pernah connect ke dApp | tidak ada approval yang bisa disalahgunakan |
| Baca `spender` dan `value` di setiap "Sign typed data" | permit phishing tidak butuh transaksi |
| Cek dan revoke approval berkala (revoke.cash) | approval tidak kedaluwarsa |
| Kontrak yang memanggil token lain: pakai `SafeERC20` | token seperti USDT tidak patuh standar |

## Coba sendiri

1. Tambahkan EIP-2612 ke token latihan dari [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md), lalu tulis router dengan
   fungsi `sweep` milik admin.
2. Tulis test Foundry untuk keempat skenario: race 100 → 50, unlimited approve lalu `sweep`, permit
   phishing (cek nonce korban tidak berubah), dan revoke.
3. Coba pakai tanda tangan permit dua kali, dan di chain ID lain. Keduanya harus revert.
4. Di fork Ethereum, coba `approve` USDT 100 → 50 langsung, lalu lewat 0 dulu. Lihat juga return
   value `approve()`.

```bash
forge test --match-path test/Approvals.t.sol -vv
anvil --fork-url https://eth.drpc.org                      # fork Ethereum untuk USDT asli
cast call 0xdAC17F958D2ee523a2206206994597C13D831ec7 'allowance(address,address)(uint256)' <owner> <spender>
```
