---
title: EVM 09.01 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 09.01 - Hasil Praktik

Hasil percobaan untuk materi [EVM 09.01 - Permit2](../EVM%2009%20-%20Detail/EVM%2009.01%20-%20Permit2.md). Dijalankan 2026-10-20: test fork mainnet
dengan USDT asli.

Project: `~/Documents/riset/evm/permit2` — `src/IPermit2.sol` (interface minimal, disalin dari repo
Uniswap), `src/Vault.sol` (contoh di materi), `test/Permit2Usdt.t.sol`. Fork Ethereum blok
26.231.000 (`eth.drpc.org`), Permit2 dan USDT asli. Jalankan: `forge test -vv`. **10/10 lulus.**

Satu-satunya approve on-chain Alice: `USDT.approve(Permit2, max)` di `setUp`. USDT tidak
mengembalikan `bool` (lihat [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md)), tapi Permit2 memakai
`SafeTransferLib`, jadi tetap jalan.

| Test | Hasil |
| --- | --- |
| Deposit 100 USDT ke vault A hanya dengan tanda tangan | ✅ vault dapat 100; `USDT.allowance(alice, vault)` tetap **0** |
| Vault B berikutnya | ✅ tanpa approve baru, cukup tanda tangan |
| Nonce 7 dulu, lalu nonce 3 | ✅ unordered nonce, urutan bebas |
| Tanda tangan dipakai ulang | ❌ `InvalidNonce` |
| Tanda tangan untuk vault A dipakai vault B | ❌ `InvalidSigner` — spender = `msg.sender` ikut di-hash |
| Deadline lewat | ❌ `SignatureExpired(deadline)` |
| **Phishing `PermitSingle` max ke attacker** | saldo Alice 1.000 → **0**, nonce transaksi Alice **tidak berubah** |
| Allowance lewat `expiration` | ❌ `AllowanceExpired` |
| `lockdown` | allowance jadi 0 → ❌ `InsufficientAllowance(0)` |

## Temuan: key test terkenal sudah didelegasi EIP-7702 di mainnet

Percobaan pertama 8/9 gagal. Trace menunjukkan Permit2 memanggil `isValidSignature` ke alamat Alice
dan revert. Alamat dari private key `0xA11CE` **punya code** di mainnet:

```
$ cast code 0xe05fcC23807536bEe418f142D19fa0d21BB0cfF7
0xef01002e623468c560072e91d7a94a1dcb2eefbdbe69cb
```

`0xef0100` + alamat = delegasi **EIP-7702** (lihat [EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)). Key
`makeAddrAndKey("alice")` dari forge-std juga sama. Private key-nya publik, jadi siapa pun (biasanya
bot sweeper) bisa memasang delegasi.

Akibatnya Permit2 memverifikasi lewat EIP-1271, bukan `ecrecover`, dan tanda tangan ECDSA biasa
ditolak. Perbaikannya: label unik untuk akun test dan `assertEq(alice.code.length, 0)` di `setUp`.
