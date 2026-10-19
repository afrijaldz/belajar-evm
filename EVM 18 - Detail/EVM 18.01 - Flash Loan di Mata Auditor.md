---
title: EVM 18.01 - Flash Loan di Mata Auditor
tags: [learning, evm, audit, flash-loan, security]
updated: 2026-10-19
---

# Flash Loan di Mata Auditor

Bagian dari [EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md)

**Flash loan adalah pinjaman tanpa jaminan yang harus kembali di transaksi yang sama.** Kalau di akhir transaksi dana + fee tidak dikembalikan, seluruh transaksi revert — seolah pinjaman tidak pernah terjadi. Karena itu pemberi pinjaman tidak menanggung risiko gagal bayar.

Ini **fitur resmi** yang disediakan protokol (Aave, Morpho, Balancer, Uniswap), bukan sesuatu yang ilegal atau hasil bobolan. Pemakaian sahnya: arbitrase antar-DEX, likuidasi posisi underwater, tukar jaminan, refinancing antar-protokol, self-liquidation. Yang menentukan sah atau tidak adalah **perbuatan di dalam callback**, bukan alatnya.

## Kenapa ini penting untuk audit

Flash loan **mematikan satu asumsi yang diam-diam dipegang banyak kontrak**: "penyerang tidak punya modal besar". Dulu, memompa harga pool atau menguasai mayoritas voting butuh jutaan dolar sungguhan — jadi dianggap tidak realistis. Flash loan membuat modal itu **gratis dan tersedia untuk siapa pun**, asal semuanya selesai dalam satu transaksi.

Jadi pertanyaan auditor berubah:

> ~~"Apakah realistis penyerang punya 40M USDT?"~~ → **"Apa yang terjadi kalau penyerang punya 40M USDT selama satu transaksi?"**

Jawabannya harus selalu dianggap: **ya, bisa.**

## Mekanika singkat

Flash loan jalan lewat **callback**, jadi peminjamnya wajib kontrak (EOA tidak bisa menerima callback). Bentuk umumnya:

```
kontrak → Lender.flashLoan(token, jumlah, data)
             Lender transfer `jumlah` ke kontrak
             Lender panggil kontrak.<callback>(..., data)   ← logika peminjam jalan di sini
             Lender cek saldo kembali ≥ jumlah + fee, kalau kurang → revert
```

Detail "kenapa harus kontrak" dan "kenapa harus satu transaksi" dibahas di Insiden BiFi - Bunga Idle Dihitung Ulang bagian "Kenapa penyerang harus bikin kontrak".

## Apa yang membuat kontrak rentan flash loan

Flash loan sendiri tidak membobol apa pun. Ia **memperbesar** bug yang sudah ada, yaitu kontrak yang mempercayai angka yang bisa digeser dalam satu transaksi:

| Angka yang dipercaya | Digeser dengan | Akibat |
| --- | --- | --- |
| Harga spot AMM (`getReserves`) | pompa pool pakai modal pinjaman | oracle salah → lender terkuras ([EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md)) |
| Utilisasi → suku bunga | deposit/borrow besar dalam satu blok | bunga salah hitung (Insiden BiFi - Bunga Idle Dihitung Ulang) |
| Rasio share/aset vault | donation + deposit (inflation attack) | penyetor pertama dicuri |
| Voting power governance | pinjam token tata kelola, voting, kembalikan | proposal jahat lolos |
| Saldo `balanceOf(this)` sebagai sumber kebenaran | transfer langsung ke kontrak | akuntansi internal meleset |

Pola umumnya sama dengan butir 11 checklist auditor di [EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md): **angka ini datang dari mana, dan siapa yang bisa menggesernya dalam satu transaksi?**

## Checklist flash-loan saat membaca kode

1. **Cari nilai yang dibaca sesaat (spot).** Harga, utilisasi, rasio share, voting power — apa pun yang diukur dari state *sekarang*. Kalau dibaca langsung tanpa penundaan waktu, anggap bisa dimanipulasi.
2. **Apakah ada penundaan waktu?** TWAP, snapshot blok lampau, checkpoint voting (`getPastVotes`) menetralkan flash loan karena nilai yang dipompa dalam satu transaksi "bertahan 0 detik".
3. **Apakah state dibaca sebelum atau sesudah input penyerang masuk?** Kalau sesudah, penyerang memilih nilainya.
4. **Apakah `balanceOf(this)` jadi sumber kebenaran?** Kalau ya, cek apakah transfer langsung (donation) merusak akuntansi.
5. **Uji dengan flash loan di test.** Pola latihan: pinjam besar di awal `test`, jalankan serangan, assert kerugian protokol. Kalau protokol tahan, invariannya terbukti. Ini yang dilatih di Damn Vulnerable DeFi (Unstoppable, Naive Receiver, Puppet) — lihat Audit 00 - Roadmap Fase 2.

## Pertahanan (sisi protokol)

- Jangan pakai harga spot; pakai **TWAP atau oracle eksternal** (Chainlink) dengan sanity check.
- Governance pakai **voting power ter-snapshot** di blok lampau, bukan saldo sekarang.
- Vault: **virtual shares/assets** untuk melawan inflation attack (pola ERC-4626 OpenZeppelin).
- Akuntansi internal pakai variabel yang di-track sendiri, bukan `balanceOf`.
- Kalau benar-benar perlu, **blokir aksi sensitif dalam transaksi yang sama** dengan deposit besar — tapi ini tambalan, bukan akar solusi.

---

Bagian dari [EVM 18 - Security Basics](../EVM%2018%20-%20Security%20Basics.md) · terkait: Insiden BiFi - Bunga Idle Dihitung Ulang
