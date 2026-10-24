---
title: EVM 22 - Monitoring
tags: [learning, evm, monitoring, events, alert, uni]
updated: 2026-10-24
---

# Monitoring

Sesi 22 dari [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md). Di sesi ini kamu menulis monitor event + alert untuk `Vault`, mengujinya di
`anvil` dengan eksploit dan reorg simulasi, lalu memakai pola yang sama di mainnet untuk mengukur laju burn UNI.
Lanjutan dari [EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md); sesi ini tidak butuh deploy sungguhan.

Hasil percobaan untuk materi ini ada di [EVM 22 - Hasil Praktik](Hasil%20Praktik/EVM%2022%20-%20Hasil%20Praktik.md).

## Monitor `Vault`

Setiap polling:

1. **Cek reorg** — hash blok terakhir yang diproses masih sama? Kalau tidak, mundur sampai cocok
   ([EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md)).
2. `eth_getLogs` untuk `Deposited` / `Withdrawn` sampai `head − confirmations`.
3. Alert **penarikan besar** (≥ ambang).
4. **Solvency on-chain**: saldo nyata vault vs `totalDeposits` per aset — hanya mungkin karena kontrak
   menyimpan `totalDeposits` (trade-off-nya di [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md)).

```bash
python3 monitor.py <rpc> <vault> <aset,...> --start <blok> --confirmations N --large-withdraw <wei> --poll 1 --max-polls 60
```

### Menguji monitor di anvil

`anvil` punya cheat RPC untuk mensimulasikan kejadian buruk tanpa eksploit sungguhan:

| Kejadian | Cara memicu | Yang harus dilaporkan monitor |
| --- | --- | --- |
| Deposit / withdraw normal | transaksi biasa | tercatat per blok |
| Penarikan besar | withdraw di atas ambang | `ALERT large withdrawal …` |
| Eksploit (saldo vault hilang) | potong saldo token vault lewat `anvil_setStorageAt` | `ALERT INSOLVENT asset … holds X but owes Y` |
| Reorg | `anvil_reorg` beberapa blok, ganti dengan transaksi lain | `ALERT reorg at block …`, lalu blok baru dibaca ulang |

Setelah reorg membatalkan blok yang berisi manipulasi storage, alert INSOLVENT harus hilang.

### Kelemahan monitor sederhana (dan perbaikannya)

- Event di blok yang belum final bisa **terlanjur dilaporkan** sebelum blok itu dibatalkan reorg → pakai
  `--confirmations N`.
- Alert yang sama **berulang tiap polling** → monitor produksi butuh deduplikasi alert.
- Output cuma `print` → produksi: kirim ke Telegram/Discord/PagerDuty, simpan state ke disk.

### Jebakan: hash dari ingatan

Topic event dan selector fungsi yang ditulis dari ingatan sering **salah**. Dengan hash salah, monitor **diam**
— tidak ada event cocok, tidak ada error. Selalu hitung dengan `cast keccak` / `cast sig`. Pola kesalahan ini
sama dengan alamat dan key di [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md).

```bash
cast keccak 'Deposited(address,address,uint256)'   # topic0 event (sesuaikan dengan signature event-mu)
cast sig 'totalDeposits(address)'                   # selector fungsi
```

## Mainnet: laju burn UNI

Pola yang sama bisa dipakai di mainnet: hitung event `Released` Firepit mainnet di N blok terakhir, lalu ubah
ke UNI per jam (setiap `Released` membakar `threshold` UNI, lihat [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md)).

Tapi `Released` di mainnet **bukan satu-satunya jalur burn**. Sejak fee switch diperluas ke L2 (Maret 2026),
tiap L2 punya Firepit sendiri (threshold **2.000 UNI**). UNI yang dibakar di L2 **di-bridge ke `0xdead` di L1**
lewat hook `_afterRelease` — yang di source Firepit dikomentari *"e.g. bridge calls"*. Jadi transfer UNI ke
`0xdead` di Ethereum datang dari banyak pengirim: Firepit mainnet, dan kontrak bridge tiap L2
(`L1StandardBridge` untuk Base/Optimism, `L1ERC20Gateway` untuk Arbitrum, `NttManager` untuk Wormhole NTT).
Withdrawal rollup OP (Base/Optimism) final ~7 hari → burn L2 tiba **terlambat dan bergelombang**.

**Pelajaran monitoring:** pantau **hasil akhir** (Transfer ke `0xdead`), bukan satu jalur mekanisme. Monitor
yang cuma melihat `Released` di mainnet bisa melewatkan sebagian besar burn.

## Coba sendiri

1. Jalankan `anvil`, deploy `Vault`, lalu jalankan `monitor.py` dengan `--large-withdraw` kecil.
2. Lakukan deposit dan withdraw besar; cek alert.
3. Potong saldo token vault dengan `anvil_setStorageAt`; cek alert INSOLVENT.
4. Panggil `anvil_reorg` 2 blok dan ganti dengan deposit dari akun lain; cek alert reorg dan apakah alert
   INSOLVENT hilang.
5. Jalankan penghitung `Released` di mainnet untuk ±1.800 blok, lalu bandingkan dengan semua transfer UNI →
   `0xdead` di periode yang sama (mis. lewat Blockscout). Kelompokkan per pengirim.
