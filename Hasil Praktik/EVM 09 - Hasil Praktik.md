---
title: EVM 09 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 09 - Hasil Praktik

Hasil percobaan untuk materi [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md). Dijalankan 2026-09-24. Empat
skenario serangan dijalankan sebagai test Foundry (7 test) terhadap token sendiri, plus perilaku
USDT di fork Ethereum mainnet.

Project: `~/Documents/riset/evm/solidity-basics` — `src/PermitToken.sol` (MyToken + EIP-2612),
`src/ShadyRouter.sol` (router dengan pintu belakang), `test/Approvals.t.sol`.

## Skenario 2: unlimited approve ke `ShadyRouter`

Alice approve `type(uint256).max` ke `ShadyRouter` untuk deposit 10 token, lalu menerima 500 token
lagi tanpa menyentuh router. Admin router memanggil `sweep` → **590 token terkuras**. Dengan approve
pas (10), `sweep` revert setelah deposit.

## Skenario 3: phishing `permit`

| | |
| --- | --- |
| Saldo Alice | 1.000 → **0** |
| Nonce transaksi Alice | **1 → 1** — tidak pernah mengirim transaksi |
| Gas | dibayar attacker |

| Test | Hasil |
| --- | --- |
| Tanda tangan dipakai dua kali | ❌ `InvalidSignature` — `nonces[owner]` sudah naik |
| Tanda tangan chain 31337 dipakai di chain 56 | ❌ `InvalidSignature` — `DOMAIN_SEPARATOR` mengikat chain ID + alamat kontrak |

## Skenario 4: revoke

`approve(spender, 0)` → `sweep` gagal, saldo aman.

## USDT di fork Ethereum (`0xdAC1…1ec7`)

| Percobaan | Hasil |
| --- | --- |
| approve 100 (dari 0) | ✅ |
| **approve 50 langsung (100 → 50)** | ❌ status 0, allowance tetap 100 |
| approve 0, lalu approve 50 | ✅ allowance 50 |
| Return value `approve()` | **`0x` (kosong)** — UNI mengembalikan `…0001` |

## Catatan

Kebiasaan "wallet hold terpisah, tidak pernah connect ke dApp" berkaitan dengan saran hold CAKE/BNB
di riset tokenomics.
