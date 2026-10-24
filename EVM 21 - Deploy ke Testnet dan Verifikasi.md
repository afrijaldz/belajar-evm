---
title: EVM 21 - Deploy ke Testnet dan Verifikasi
tags: [learning, evm, foundry, deploy, testnet, verification]
updated: 2026-10-24
---

# Deploy ke Testnet dan Verifikasi

Sesi 21 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu menyiapkan wallet deployer, memilih RPC testnet, menulis
script deploy, dan memahami cara explorer memverifikasi source kontrak. Lanjutan dari
[EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md).

Hasil percobaan untuk materi ini ada di [EVM 21 - Hasil Praktik](Hasil%20Praktik/EVM%2021%20-%20Hasil%20Praktik.md).

## Wallet deployer

Pakai wallet baru khusus untuk deploy latihan, terpisah dari wallet yang memegang dana. Catatan wallet ini ada
di EVM Testnet Deployer (`credential/`). Address deployer: `0x54f4fAA9c7e2476Cb248C8E8c7bb6F260e4dbb56`.

Key **tidak** disimpan di folder project (tidak ada `.env`). Script membaca `DEPLOYER_KEY` dari environment;
perintah untuk mengisinya ada di catatan kredensial itu. Setelah mengisi, cek bahwa alamat hasil turunan key
cocok dengan address deployer.

Wallet baru saldonya 0. Isi dulu dari faucet testnet (Sepolia / BSC testnet) sebelum deploy sungguhan.

## RPC testnet

RPC publik gratis kualitasnya berbeda-beda: ada yang cepat, ada yang lambat, ada yang jalan lalu timeout
beberapa menit kemudian, ada yang langsung menolak (HTTP 400/404). Uji beberapa RPC dulu, lalu simpan yang
stabil di `[rpc_endpoints]` `foundry.toml`. Kandidat yang bisa diuji:

- Sepolia: `sepolia.gateway.tenderly.co`, `1rpc.io/sepolia`, `ethereum-sepolia-rpc.publicnode.com`,
  `sepolia.drpc.org`, `rpc.sepolia.org`
- BSC testnet (chain 97): `bsc-testnet-rpc.publicnode.com`

```bash
cast chain-id --rpc-url https://<rpc>     # cepat cek RPC hidup dan chain-nya benar
```

## Script deploy

Contoh `DeployVault.s.sol`: deploy `Vault` + `MockToken`, mint, approve, deposit/withdraw token,
deposit/withdraw 0,001 ETH — supaya explorer menampilkan aktivitas.

```bash
export DEPLOYER_KEY=...   # lihat [[EVM Testnet Deployer]]
forge script script/DeployVault.s.sol --rpc-url sepolia               # simulasi, tidak mengirim apa pun
forge script script/DeployVault.s.sol --rpc-url sepolia --broadcast   # kirim sungguhan
```

### Simulasi dulu

Tanpa `--broadcast`, `forge script` menjalankan semua langkah di atas state chain sungguhan tapi tidak mengirim
apa pun. Dengan saldo 0, langkah yang tidak butuh ETH (deploy, mint, approve, token) tetap lolos, dan langkah
yang mengirim ETH berhenti dengan `OutOfFunds`. Alamat kontrak sudah pasti sebelum deploy, karena ditentukan
oleh deployer + nonce ([EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md)): kontrak pertama di nonce 0, kedua di nonce 1, dan seterusnya.

### Menghitung biaya di fork

Untuk tahu berapa ETH yang perlu diminta dari faucet, jalankan script di **fork Sepolia lokal** dengan saldo
palsu. Hasilnya: jumlah transaksi, total gas, dan biaya. Estimasi gas price forge biasanya lebih tinggi dari
gas price nyata (sudah termasuk margin), jadi biaya sebenarnya lebih kecil dari estimasinya.

**Jebakan:** folder `broadcast/DeployVault.s.sol/<chain-id>/` juga terisi oleh run di fork, karena fork memakai
chain ID asli. Isinya bukan bukti deploy sungguhan. Deploy sungguhan akan menimpa `run-latest.json`.

## Verifikasi: prinsipnya

Explorer meng-compile ulang source dengan **pengaturan yang sama persis**, lalu membandingkan dengan bytecode
di chain.

- Bytecode on-chain harus identik dengan `forge inspect … deployedBytecode`, termasuk metadata.
- Di ujung bytecode ada **metadata CBOR**: hash IPFS dari source + pengaturan. Kalau cocok, Sourcify menyebutnya
  "full match".
- Pengaturan berbeda (mis. `optimizer_runs` 200 → 10.000) menghasilkan bytecode berbeda, dan verifikasi gagal.

Pengaturan yang harus sama dengan saat deploy: versi solc, optimizer on/off, `optimizer_runs`, `via_ir`.
Contoh: solc **0.8.28**, optimizer on, **runs 200**, `via_ir` off.

**Jebakan:** `forge inspect --optimizer-runs 10000` **menimpa artefak di `out/`**. Setelah bereksperimen,
`forge build --force` sebelum verifikasi.

## Tugas: deploy sungguhan setelah saldo masuk

1. Cek saldo deployer di testnet.
2. `forge build --force`, lalu jalankan script dengan `--broadcast`.
3. Verifikasi lewat Sourcify dan Blockscout (keduanya tanpa API key).

```bash
cast balance <deployer> --rpc-url https://sepolia.gateway.tenderly.co
forge build --force
forge script script/DeployVault.s.sol --rpc-url sepolia --broadcast
# verifikasi tanpa API key:
forge verify-contract <alamat-vault> src/Vault.sol:Vault --chain sepolia --verifier sourcify
forge verify-contract <alamat-vault> src/Vault.sol:Vault --chain sepolia \
  --verifier blockscout --verifier-url https://eth-sepolia.blockscout.com/api/
```

Alamat yang dihitung lewat nonce hanya berlaku kalau deploy adalah transaksi pertama wallet (nonce 0). Kalau
tidak, pakai alamat dari output `forge script`.
