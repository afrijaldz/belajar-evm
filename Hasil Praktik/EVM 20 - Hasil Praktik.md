---
title: EVM 20 - Hasil Praktik
tags: [learning, evm, hasil-praktik]
updated: 2026-10-24
---

# EVM 20 - Hasil Praktik

Hasil percobaan untuk materi [EVM 20 - Gas Optimization](../EVM%2020%20-%20Gas%20Optimization.md). Dijalankan 2026-09-24. Project:
`~/Documents/riset/evm/vault-project` — `bench.py` (benchmark transaksi nyata di anvil port 8546),
`.gas-snapshot` (baseline `forge snapshot`).

```bash
cd ~/Documents/riset/evm/vault-project
python3 bench.py
forge snapshot --no-match-contract VaultInvariantTest --diff
forge test --gas-report
```

## Hasil (gas transaksi nyata)

| Langkah | Deploy | `depositETH` pertama | `depositETH` lagi | `withdraw` ETH | `depositToken` pertama | `depositToken` lagi | `withdraw` token |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 baseline (optimizer off) | 916.426 | 68.125 | 33.925 | 42.086 | 112.620 | 61.320 | 52.521 |
| 1 **optimizer on** | 511.255 (−44%) | 67.668 | 33.468 | 41.172 | 109.003 | 57.703 | 50.497 |
| 2 `abi.encodeCall` | **sama persis** | | | | | | |
| 3 `unchecked` (2 pengurangan) | 503.913 | 67.668 | 33.468 | 40.998 | 109.003 | 57.703 | 50.323 |
| (4) tanpa `totalDeposits` — **tidak dipakai** | 460.645 | **45.424 (−33%)** | 28.324 | 35.904 | 86.714 | 52.514 | 45.229 |
| (5) `via_ir` — **tidak dipakai** | **385.220 (−58%)** | 67.611 | 33.411 | 40.822 | 108.133 | 56.833 | 49.859 |

`forge snapshot --diff` setelah langkah 1–3: 17/17 test lebih murah, total −23% (termasuk deploy di `setUp`).

## Angka per pelajaran

- Optimizer: deploy −44%; runtime cuma terpotong 1–6%.
- `abi.encodeCall`: selisih 0 gas.
- `unchecked`: hemat 174 gas per `withdraw`.
- Menghapus `totalDeposits`: −22.244 gas pada deposit pertama. **Keputusan: `totalDeposits` dipertahankan**
  demi auditabilitas solvency on-chain (dipakai monitor di [EVM 22 - Monitoring](../EVM%2022%20-%20Monitoring.md)).
- `via_ir`: deploy −24% lagi dibanding langkah 3, runtime cuma beberapa ratus gas. Tidak dipakai.
