---
title: EVM 21 - Deploy ke Testnet dan Verifikasi
tags: [learning, evm, foundry, deploy, testnet, verification]
updated: 2026-09-24
---

# Deploy ke Testnet dan Verifikasi

Catatan dari Sesi 21 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. **Status: persiapan selesai, deploy sungguhan
menunggu saldo faucet.** Lanjutan dari [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md).

Project: `~/Documents/riset/evm/vault-project` — `script/DeployVault.s.sol`, `[rpc_endpoints]` di
`foundry.toml`.

## Wallet deployer

Wallet baru khusus modul ini, dicatat di EVM Testnet Deployer (`credential/`).

| | |
| --- | --- |
| Address | `0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56` |
| Saldo 2026-09-24 | 0 di Sepolia dan BSC testnet |

Key **tidak** disimpan di folder project (tidak ada `.env`). Script membaca `DEPLOYER_KEY` dari environment;
perintah untuk mengisinya dari catatan kredensial ada di catatan itu — sudah diuji: alamat hasil
turunannya cocok.

## RPC testnet (2026-09-24)

| RPC | Hasil |
| --- | --- |
| `sepolia.gateway.tenderly.co` | ✅ 539 ms — dipakai (`sepolia` di `foundry.toml`) |
| `1rpc.io/sepolia` | ✅ 2,7 s |
| `ethereum-sepolia-rpc.publicnode.com` | ⚠️ jalan lalu timeout 12 s beberapa menit kemudian |
| `sepolia.drpc.org` | ❌ HTTP 400 |
| `rpc.sepolia.org` | ❌ HTTP 404 |
| `bsc-testnet-rpc.publicnode.com` | ✅ (chain 97) |

## Script deploy

`DeployVault.s.sol`: deploy `Vault` + `MockToken`, mint, approve, deposit/withdraw token, deposit/withdraw
0,001 ETH — supaya explorer menampilkan aktivitas.

```bash
cd ~/Documents/riset/evm/vault-project
export DEPLOYER_KEY=...   # lihat [[EVM Testnet Deployer]]
forge script script/DeployVault.s.sol --rpc-url sepolia               # simulasi, tidak mengirim apa pun
forge script script/DeployVault.s.sol --rpc-url sepolia --broadcast   # kirim sungguhan
```

### Simulasi di Sepolia sungguhan (saldo 0)

Deploy, mint, approve, deposit/withdraw token **lolos**; berhenti di `depositETH` dengan `OutOfFunds` —
sesuai harapan. Alamat sudah pasti sebelum deploy (deployer + nonce, [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md)):

| Kontrak | Alamat (nonce) |
| --- | --- |
| Vault | `0x31037DeE511BAB55A5180F77667aDbAfF6d9993C` (0) |
| MockToken | `0x2262c8D4294a413fCFfca8A3ab05cb7d38c06A93` (1) |

### Run penuh di fork Sepolia lokal (saldo palsu)

| | |
| --- | --- |
| Transaksi | 8, semua sukses |
| Total gas | 1.189.622 |
| Gas price Sepolia | ~1,09 gwei (estimasi forge 1,94 gwei + margin) |
| Biaya | ~0,0013 ETH (estimasi forge 0,0031) |
| **Butuh dari faucet** | **≥ 0,005 Sepolia ETH** |

Folder `broadcast/DeployVault.s.sol/11155111/` berisi hasil run **fork** (fork memakai chain ID Sepolia),
bukan deploy sungguhan. Deploy sungguhan menimpa `run-latest.json`.

## Verifikasi: prinsipnya

Explorer meng-compile ulang source dengan **pengaturan yang sama persis**, lalu membandingkan dengan bytecode
di chain. Diuji di fork:

| | Hasil |
| --- | --- |
| Bytecode on-chain vs `forge inspect … deployedBytecode` | **identik**, 2.084 byte, termasuk metadata |
| Metadata CBOR di ujung bytecode | hash IPFS dari source + pengaturan → cocok = "full match" Sourcify |
| Kalau `optimizer_runs` 200 → 10.000 | **berbeda** (2.759 byte) → verifikasi akan gagal |

Pengaturan yang harus sama: solc **0.8.28**, optimizer on, **runs 200**, `via_ir` off.

**Jebakan:** `forge inspect --optimizer-runs 10000` **menimpa artefak di `out/`**. Setelah bereksperimen,
`forge build --force` sebelum verifikasi.

## Langkah setelah saldo masuk

```bash
cast balance 0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56 --rpc-url https://sepolia.gateway.tenderly.co
forge build --force
forge script script/DeployVault.s.sol --rpc-url sepolia --broadcast
# verifikasi tanpa API key:
forge verify-contract 0x31037DeE511BAB55A5180F77667aDbAfF6d9993C src/Vault.sol:Vault --chain sepolia --verifier sourcify
forge verify-contract 0x31037DeE511BAB55A5180F77667aDbAfF6d9993C src/Vault.sol:Vault --chain sepolia \
  --verifier blockscout --verifier-url https://eth-sepolia.blockscout.com/api/
```

Alamat di atas hanya berlaku kalau deploy adalah transaksi pertama wallet (nonce 0). Kalau tidak, pakai
alamat dari output `forge script`.
