---
title: EVM 23 - Multi-Chain
tags: [learning, evm, multichain, create2, bridge, l2]
updated: 2026-10-24
---

# Multi-Chain

Sesi 23 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu membandingkan beberapa chain EVM berdampingan (Ethereum,
BSC, Base, Arbitrum), men-deploy kontrak yang sama ke dua fork dengan CREATE dan CREATE2, menguji replay antar
chain, dan memeriksa bridge UNI ke L2. Lanjutan dari [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md).

Hasil percobaan untuk materi ini ada di [EVM 23 - Hasil Praktik](Hasil%20Praktik/EVM%2023%20-%20Hasil%20Praktik.md).

## Yang sama dan yang beda antar chain

**Sama:** kontrak infrastruktur tertentu ada **di alamat yang sama** di banyak chain EVM — fondasi deploy
multi-chain:

- Multicall3 di `0xcA11bde05977b3631167028862bE2a173976CA11`
- Deployer CREATE2 Arachnid di `0x4e59b44847b379578588920cA78FbF26c0B4956C`

**Beda:** chain ID, block time, dan arti `block.number`.

- Block time sangat bervariasi: sekitar 12 s di Ethereum, 2 s di Base, di bawah 1 s di BSC dan Arbitrum.
- **Di Arbitrum, `block.number` di dalam kontrak = nomor blok Ethereum (L1)**, bukan nomor blok L2 yang
  dikembalikan `eth_blockNumber`. Cara melihatnya: bandingkan `eth_blockNumber` dengan
  `Multicall3.getBlockNumber()`.

### Bahaya `block.number` untuk waktu

Kunci "100.000 blok" dengan kode yang identik: di BSC habis dalam ≈ **12,5 jam**, di Ethereum dan Arbitrum
≈ **14 hari** (Arbitrum ikut blok L1). Kontrak multi-chain memakai `block.timestamp` untuk mengukur waktu.

## CREATE vs CREATE2

- **CREATE**: alamat = fungsi dari deployer + nonce ([EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md)). Wallet yang sama bisa punya
  nonce berbeda di tiap chain, jadi kontrak yang sama mendarat di **alamat berbeda**.
- **CREATE2**: alamat = fungsi dari deployer, salt, dan bytecode. Lewat deployer Arachnid (alamatnya sama di
  semua chain) dengan salt dan bytecode yang sama, kontrak mendarat di **alamat yang sama** di semua chain.

```
alamat CREATE2 = keccak256(0xff ‖ deployer ‖ salt ‖ keccak256(initCode))[12:]
cast create2 --deployer 0x4e59b44847b379578588920cA78FbF26c0B4956C --salt <salt> --init-code <bytecode>
```

Alamat CREATE2 bisa dihitung **sebelum** menyentuh chain apa pun. Syarat: bytecode identik byte per byte —
termasuk metadata; `optimizer_runs` berbeda → alamat berbeda ([EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md)).
Deployer Arachnid menerima calldata = `salt ‖ initCode`.

## Replay antar chain

Chain ID ikut ditandatangani (EIP-155, [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md)). Transaksi yang ditandatangani untuk
satu chain diterima di chain itu, tapi ditolak di chain lain dengan `invalid chain id for signer`.

## Bridge: UNI di L2 (lock-and-mint)

UNI asli dikunci di kontrak bridge L1 (escrow); versi L2 dicetak. Invariant bridge sehat:
**UNI terkunci di L1 ≥ supply UNI di L2**.

Token L2 menyimpan alamat token asalnya di L1, dan nama fungsinya beda per bridge:

| L2 | Bridge L1 | Fungsi asal-token |
| --- | --- | --- |
| Base | `L1StandardBridge` | `remoteToken()` |
| Optimism | `L1StandardBridge` | `l1Token()` |
| Arbitrum | `L1ERC20Gateway` | `l1Address()` |

Escrow L1 bisa **lebih besar** dari supply L2. Penyebab yang paling mungkin adalah **burn yang sedang dalam
perjalanan**: burn di L2 langsung mengurangi supply L2, tapi escrow L1 baru dilepas ke `0xdead` setelah
withdrawal final (~7 hari untuk rollup OP, lihat [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md)). Sebagian kelebihan bisa juga UNI
yang terkirim langsung ke bridge — keduanya tidak bisa dipisahkan dari data saldo saja.

## Checklist kontrak multi-chain

| Hal | Kenapa |
| --- | --- |
| Waktu pakai `block.timestamp`, bukan `block.number` | block time 0,27–12 s, Arbitrum `block.number` = L1 |
| Deploy via CREATE2 dengan salt + bytecode sama | alamat sama di semua chain |
| Kunci pengaturan compiler | beda bytecode = beda alamat CREATE2 + verifikasi gagal |
| Jangan asumsikan token sama = alamat sama | UNI punya alamat berbeda di tiap L2 |
| Cek `evm_version` yang didukung chain target | opcode baru belum tentu ada di semua chain |
| Pantau invariant bridge (escrow ≥ supply L2) | bridge adalah titik kepercayaan terbesar |

## Coba sendiri

1. Untuk tiap chain, baca chain ID, block time, code size Multicall3 dan deployer Arachnid, lalu bandingkan
   `eth_blockNumber` dengan `Multicall3.getBlockNumber()`.
2. Jalankan fork Base (port 8550) dan fork BSC (port 8551). Deploy `Vault` dari wallet baru yang sama dengan
   CREATE; di salah satu chain kirim satu transaksi lain dulu supaya nonce-nya beda. Bandingkan alamatnya.
3. Deploy lagi lewat deployer Arachnid dengan salt yang sama; bandingkan alamat dan hasil `cast create2`.
4. Tanda tangani transaksi untuk chain 8453, lalu kirim ke fork Base dan fork BSC.
5. Untuk tiap L2, baca supply UNI L2, alamat asal-tokennya, dan saldo UNI di escrow L1. Cek invariant bridge.

```bash
cast chain-id --rpc-url <rpc>
cast code 0xcA11bde05977b3631167028862bE2a173976CA11 --rpc-url <rpc> | wc -c
cast call 0xcA11bde05977b3631167028862bE2a173976CA11 'getBlockNumber()(uint256)' --rpc-url <rpc>
cast block-number --rpc-url <rpc>
anvil --fork-url <rpc-base> --port 8550
anvil --fork-url <rpc-bsc> --port 8551
```
