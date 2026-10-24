---
title: EVM 21 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 21 - Hasil Praktik

Hasil percobaan untuk materi [EVM 21 - Deploy ke Testnet dan Verifikasi](../EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md). Dijalankan 2026-09-24. **Status:
persiapan selesai, deploy sungguhan menunggu saldo faucet.** Project: `~/Documents/riset/evm/vault-project` —
`script/DeployVault.s.sol`, `[rpc_endpoints]` di `foundry.toml`.

## Wallet deployer

Wallet baru khusus modul ini, dicatat di EVM Testnet Deployer (`credential/`).

| | |
| --- | --- |
| Address | `0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56` |
| Saldo 2026-09-24 | 0 di Sepolia dan BSC testnet |

Perintah pengisian `DEPLOYER_KEY` dari catatan kredensial sudah diuji: alamat hasil turunannya cocok.

## RPC testnet (2026-09-24)

| RPC | Hasil |
| --- | --- |
| `sepolia.gateway.tenderly.co` | ✅ 539 ms — dipakai (`sepolia` di `foundry.toml`) |
| `1rpc.io/sepolia` | ✅ 2,7 s |
| `ethereum-sepolia-rpc.publicnode.com` | ⚠️ jalan lalu timeout 12 s beberapa menit kemudian |
| `sepolia.drpc.org` | ❌ HTTP 400 |
| `rpc.sepolia.org` | ❌ HTTP 404 |
| `bsc-testnet-rpc.publicnode.com` | ✅ (chain 97) |

## Simulasi di Sepolia sungguhan (saldo 0)

Deploy, mint, approve, deposit/withdraw token **lolos**; berhenti di `depositETH` dengan `OutOfFunds` —
sesuai harapan. Alamat sudah pasti sebelum deploy:

| Kontrak | Alamat (nonce) |
| --- | --- |
| Vault | `0x31037DeE511BAB55A5180F77667aDbAfF6d9993C` (0) |
| MockToken | `0x2262c8D4294a413fCFfca8A3ab05cb7d38c06A93` (1) |

## Run penuh di fork Sepolia lokal (saldo palsu)

| | |
| --- | --- |
| Transaksi | 8, semua sukses |
| Total gas | 1.189.622 |
| Gas price Sepolia | ~1,09 gwei (estimasi forge 1,94 gwei + margin) |
| Biaya | ~0,0013 ETH (estimasi forge 0,0031) |
| **Butuh dari faucet** | **≥ 0,005 Sepolia ETH** |

Folder `broadcast/DeployVault.s.sol/11155111/` berisi hasil run **fork** (fork memakai chain ID Sepolia),
bukan deploy sungguhan.

## Verifikasi, diuji di fork

| | Hasil |
| --- | --- |
| Bytecode on-chain vs `forge inspect … deployedBytecode` | **identik**, 2.084 byte, termasuk metadata |
| Metadata CBOR di ujung bytecode | hash IPFS dari source + pengaturan → cocok = "full match" Sourcify |
| Kalau `optimizer_runs` 200 → 10.000 | **berbeda** (2.759 byte) → verifikasi akan gagal |

Pengaturan project: solc 0.8.28, optimizer on, runs 200, `via_ir` off.

## Perintah yang tertunda (setelah saldo masuk)

```bash
cd ~/Documents/riset/evm/vault-project
cast balance 0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56 --rpc-url https://sepolia.gateway.tenderly.co
forge build --force
forge script script/DeployVault.s.sol --rpc-url sepolia --broadcast
forge verify-contract 0x31037DeE511BAB55A5180F77667aDbAfF6d9993C src/Vault.sol:Vault --chain sepolia --verifier sourcify
forge verify-contract 0x31037DeE511BAB55A5180F77667aDbAfF6d9993C src/Vault.sol:Vault --chain sepolia \
  --verifier blockscout --verifier-url https://eth-sepolia.blockscout.com/api/
```

Alamat `0x3103…993C` hanya berlaku kalau deploy adalah transaksi pertama wallet (nonce 0).
