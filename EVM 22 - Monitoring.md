---
title: EVM 22 - Monitoring
tags: [learning, evm, monitoring, events, alert, uni]
updated: 2026-09-24
---

# Monitoring

Catatan dari Sesi 22 [EVM 00 - Roadmap](EVM%2000%20-%20Roadmap.md), 2026-09-24. Monitor event + alert untuk `Vault` diuji di `anvil`
dengan eksploit dan reorg simulasi, lalu pola yang sama dipakai di mainnet untuk laju burn UNI — dan
membongkar koreksi untuk riset. Lanjutan dari [EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md) (Sesi 21 masih
menunggu faucet; sesi ini tidak butuh deploy sungguhan).

Kode: `~/Documents/riset/evm/monitor/` — `monitor.py` (Vault), `uni_burn_rate.py` (mainnet).

## Monitor `Vault`

Setiap polling:

1. **Cek reorg** — hash blok terakhir yang diproses masih sama? Kalau tidak, mundur sampai cocok
   ([EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md)).
2. `eth_getLogs` untuk `Deposited` / `Withdrawn` sampai `head − confirmations`.
3. Alert **penarikan besar** (≥ ambang).
4. **Solvency on-chain**: saldo nyata vault vs `totalDeposits` per aset — hanya mungkin karena
   `totalDeposits` dipertahankan di [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md).

```bash
python3 monitor.py <rpc> <vault> <aset,...> --start <blok> --confirmations N --large-withdraw <wei> --poll 1 --max-polls 60
```

### Uji di anvil

| Kejadian | Output monitor |
| --- | --- |
| Deposit / withdraw normal | tercatat per blok |
| Withdraw 120 token | `ALERT large withdrawal 120.0000 by 0x7099… in block 8` |
| Saldo token vault dipotong lewat `anvil_setStorageAt` (simulasi eksploit) | `ALERT INSOLVENT asset 0xe7f1… holds 30.0000 but owes 70.0000` |
| `anvil_reorg` 2 blok, diganti deposit 3 ETH akun #5 | `ALERT reorg at block 10`, `… block 9`, lalu blok 9 baru dibaca |

Setelah reorg, alert INSOLVENT hilang: manipulasi storage ada di blok yang dibatalkan.

### Kelemahan yang terlihat (dan perbaikannya)

- Deposit di blok 10 **sudah terlanjur dilaporkan** sebelum blok itu dibatalkan → pakai `--confirmations N`.
- Alert INSOLVENT **berulang tiap polling** → monitor produksi butuh deduplikasi alert.
- Output cuma `print` → produksi: kirim ke Telegram/Discord/PagerDuty, simpan state ke disk.

### Jebakan: hash dari ingatan

Topic `Deposited` dan selector `totalDeposits` yang ditulis dari ingatan **salah** (`0x2da4…`, `0xccd6e1e5`;
benar `0x8752…68a7`, `0xe9403256`). Dengan hash salah, monitor **diam** — tidak ada event cocok, tidak ada
error. Semua hash sekarang dari `cast keccak` / `cast sig`. Pola kesalahan sama dengan alamat dan key di
Sesi 11.

## Mainnet: laju burn UNI

`uni_burn_rate.py`: hitung event `Released` Firepit mainnet di N blok terakhir.

| Jendela | Released | UNI |
| --- | ---: | ---: |
| 1.800 blok (6,07 jam) | **1** (nonce 1.360) | 4.000 → **659 UNI/jam** |

Tidak cocok dengan +68.000 UNI "dalam beberapa jam" di [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md). Semua
transfer UNI → `0xdead` sejak 2026-09-23 06:00 UTC (Blockscout):

| Pengirim | Kontrak | Asal | UNI |
| --- | --- | --- | ---: |
| `0x85001cc4867c5e1c22da4b79bb8852b9e2a06da0` | `L1ERC20Gateway` (proxy) | gateway gaya Arbitrum, chain belum diidentifikasi | 60.000 (30 tx) |
| `0x3154cf16ccdb4c6d922629664174b904d80f2c35` | `L1StandardBridge` | **Base** | 20.000 |
| `0xc6208ce00b3f88c6555833ab9bbc943717dfd941` | kontrak tanpa nama | belum diidentifikasi | 12.000 |
| `0xa3a7b6f88361f48403514059f1f16c8e78d60eec` | `L1ERC20Gateway` | **Arbitrum** | 8.000 |
| `0x6569925aac77d6b8bb085f31f9828ff80d5a0c44` | `NttManager` | Wormhole NTT | 6.000 |
| `0x9b715efb4ebdb14862d079a841ba4983ffd69236` | bot Sesi 6 | Firepit **Ethereum** | 4.000 |
| `0x99c9fc46f92e8a1c0dec1b1747d010903e884be1` | `L1StandardBridge` | **Optimism** | 2.000 |
| **Total** | 51 transfer | | **112.000** |

**Penjelasan:** sejak fee switch diperluas ke L2 (Maret 2026), tiap L2 punya Firepit sendiri (threshold
**2.000 UNI**, paket 2.000 di atas). UNI yang dibakar di L2 **di-bridge ke `0xdead` di L1** lewat hook
`_afterRelease` — yang di source Firepit dikomentari *"e.g. bridge calls"*. Withdrawal rollup OP
(Base/Optimism) final ~7 hari → burn L2 tiba **terlambat dan bergelombang**.

Firepit mainnet cuma **~4%** burn dalam sampel ini.

**Pelajaran monitoring:** pantau **hasil akhir** (Transfer ke `0xdead`), bukan satu jalur mekanisme.
Monitor yang cuma melihat `Released` di mainnet melewatkan ~96% burn.

Koreksi sudah ditambahkan ke [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md) dan
UNI Diperiksa dengan Saringan Lima Chain.
