---
title: EVM 09 - Bahaya Approval ERC-20
tags: [learning, evm, solidity, erc20, security]
updated: 2026-09-24
---

# Bahaya Approval ERC-20

Catatan dari Sesi 9 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Empat skenario serangan dijalankan sebagai test
Foundry (7 test) terhadap token sendiri, plus perilaku USDT di fork Ethereum mainnet. Lanjutan dari
[EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md).

Project: `~/Documents/riset/evm/solidity-basics` — `src/PermitToken.sol` (MyToken + EIP-2612),
`src/ShadyRouter.sol` (router dengan pintu belakang), `test/Approvals.t.sol`.

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

Alice approve `type(uint256).max` ke `ShadyRouter` hanya untuk deposit 10 token. Berbulan-bulan
kemudian dia menerima 500 token lagi, tidak pernah menyentuh router. Admin router memanggil
`sweep` → **590 token terkuras**, termasuk yang datang **setelah** approve.

- **Approval tidak pernah kedaluwarsa** dan berlaku untuk saldo masa depan.
- Kasus yang sama terjadi kalau router jujur tapi kunci admin-nya dicuri, atau ada bug.
- **Approve pas (10)** → setelah deposit allowance 0 → `sweep` revert. Kerusakan terbatas.

## Skenario 3: phishing tanda tangan `permit` (EIP-2612)

`permit` = approve lewat **tanda tangan off-chain**. Siapa pun yang memegang tanda tangan bisa
mengirimkannya.

Alice menandatangani "pesan login" yang sebenarnya permit unlimited untuk attacker:

| | |
| --- | --- |
| Saldo Alice | 1.000 → **0** |
| Nonce transaksi Alice | **1 → 1** — tidak pernah mengirim transaksi |
| Gas | dibayar attacker |

Korban tidak melihat "Approve" di wallet, tidak bayar gas. Yang muncul hanya permintaan **"Sign
typed data"** berisi `spender`, `value`, `deadline` — field itu yang wajib dibaca sebelum tanda
tangan.

Proteksi yang terbukti bekerja:

| Test | Hasil |
| --- | --- |
| Tanda tangan dipakai dua kali | ❌ `InvalidSignature` — `nonces[owner]` sudah naik |
| Tanda tangan chain 31337 dipakai di chain 56 | ❌ `InvalidSignature` — `DOMAIN_SEPARATOR` mengikat chain ID + alamat kontrak |

Anatomi permit (EIP-712):

```
digest = keccak256("\x19\x01" ‖ DOMAIN_SEPARATOR ‖ keccak256(PERMIT_TYPEHASH, owner, spender, value, nonce, deadline))
signer = ecrecover(digest, v, r, s)   → harus == owner
```

## Skenario 4: revoke

`approve(spender, 0)` → `sweep` gagal, saldo aman. **Revoke melindungi masa depan, bukan masa
lalu** — token yang sudah ditarik tidak kembali.

## Token nyata: USDT (fork Ethereum, `0xdAC1…1ec7`)

| Percobaan | Hasil |
| --- | --- |
| approve 100 (dari 0) | ✅ |
| **approve 50 langsung (100 → 50)** | ❌ status 0, allowance tetap 100 |
| approve 0, lalu approve 50 | ✅ allowance 50 |
| Return value `approve()` | **`0x` (kosong)** — UNI mengembalikan `…0001` |

- **USDT memaksa aturan "nol dulu"** sebagai mitigasi race. Tapi ini **tidak sepenuhnya
  mencegah**: Bob tetap bisa menghabiskan 100 sebelum `approve(0)` masuk. Aturan ini hanya memberi
  kesempatan mengecek di antara dua langkah.
- **USDT tidak mengembalikan `bool`**, melanggar EIP-20. Kontrak dengan
  `require(token.approve(...))` akan revert dengan USDT karena tidak ada yang bisa di-decode.
  Inilah alasan `SafeERC20` (OpenZeppelin) ada. USDT dibuat sebelum EIP-20 final dan tidak bisa
  diubah.

## Pelajaran praktis

| Kebiasaan | Kenapa |
| --- | --- |
| Approve jumlah pas, bukan unlimited | membatasi kerusakan skenario 2 |
| Wallet hold terpisah, tidak pernah connect ke dApp | tidak ada approval yang bisa disalahgunakan (lihat saran hold CAKE/BNB) |
| Baca `spender` dan `value` di setiap "Sign typed data" | permit phishing tidak butuh transaksi |
| Cek dan revoke approval berkala (revoke.cash) | approval tidak kedaluwarsa |
| Kontrak yang memanggil token lain: pakai `SafeERC20` | token seperti USDT tidak patuh standar |
