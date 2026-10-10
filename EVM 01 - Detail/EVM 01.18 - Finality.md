---
title: EVM 01.18 - Finality
tags: [learning, evm, ethereum, bsc]
updated: 2026-10-10
---

# Finality

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)

Pertanyaan: finality itu gimana cara kerjanya?

**Blok yang sudah *final* tidak bisa dibatalkan, kecuali minimal 1/3 total stake rela kena
slashing.** Sebelum final, blok masih bisa diganti cabang lain (*reorg*). Finality adalah
titik di mana "hampir pasti" berubah jadi "pasti, kecuali ada yang mau membayar sangat mahal".

## Tiga tingkat kepastian

RPC `eth_*` menyediakan tiga tag blok, sesuai tingkat kepastiannya:

| Tag | Artinya | Bisa berubah? |
| --- | --- | --- |
| `latest` | ujung rantai menurut node ini | ya, bisa kena reorg |
| `safe` | blok checkpoint yang sudah **justified** | sangat kecil kemungkinannya |
| `finalized` | blok checkpoint yang sudah **finalized** | tidak, tanpa 1/3 stake kena slashing |

Contoh pakai: `cast block finalized --rpc-url …`. Exchange dan bridge biasanya menunggu
`finalized` sebelum menganggap deposit masuk (pengetahuan umum).

## Cara kerjanya: checkpoint, justified, finalized

Mekanismenya bernama **Casper FFG**, dan bahannya adalah `source` dan `target` di setiap
attestation (lihat [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)).

1. **Checkpoint** = blok pertama setiap epoch.
2. Setiap attestation berisi satu "link" dari checkpoint `source` (yang sudah justified) ke
   checkpoint `target` (epoch ini).
3. **Justified:** kalau link ke sebuah checkpoint didukung validator dengan total stake
   minimal **2/3**, checkpoint itu jadi justified.
4. **Finalized:** kalau checkpoint justified di epoch N, lalu checkpoint epoch N+1 juga jadi
   justified dengan `source` = checkpoint N, maka checkpoint N jadi **finalized**. Semua blok
   sebelum checkpoint itu ikut final.
5. Hitungannya dilakukan di **batas epoch**, bukan di setiap blok.

```text
            epoch N             epoch N+1            epoch N+2
          ┌──────────┐        ┌──────────┐         ┌──────────┐
          │ CP N ... │        │ CP N+1...│         │ CP N+2...│
          └──────────┘        └──────────┘         └──────────┘
 suara:   source N-1 → N      source N → N+1
 di akhir epoch N:   CP N justified
 di akhir epoch N+1: CP N+1 justified  → CP N finalized
```

Akibatnya, blok di awal epoch N jadi final sekitar 2 epoch kemudian (±12,8 menit). Blok yang
sedikit setelah checkpoint harus menunggu checkpoint berikutnya, jadi bisa sampai ±19 menit.
Rentang ini diturunkan dari aturan di atas; yang diukur langsung cuma posisi checkpoint (lihat
Bukti).

## Kenapa harus 2/3

Supaya dua blok final yang bertentangan bisa terjadi, dua kelompok yang masing-masing berisi
2/3 stake harus mendukung dua cabang berbeda. Dua kelompok 2/3 pasti beririsan minimal 1/3.
Validator di irisan itu berarti menandatangani dua suara yang bertentangan, dan itu **double
vote atau surround vote** yang bisa dibuktikan (lihat [EVM 01.16 - Slashing](EVM%2001.16%20-%20Slashing.md)). Jadi
membatalkan finality pasti memakan minimal 1/3 total stake.

## Kalau finality macet

Kalau validator yang online kurang dari 2/3 stake, rantai **tetap membuat blok**, tapi tidak
ada checkpoint baru yang final. Setelah 4 epoch tanpa finality, *inactivity leak* mulai
mengurangi stake validator yang offline sedikit demi sedikit, sampai validator yang online
kembali mencapai 2/3 dan finality jalan lagi (pengetahuan umum).

Ethereum mainnet pernah kehilangan finality dua kali pada Mei 2023, masing-masing kurang dari
sekitar satu jam, karena masalah di beberapa consensus client. Blok tetap jalan selama
kejadian itu (pengetahuan umum, tidak dicek ulang).

## BSC

BSC punya *fast finality* (BEP-126): validator memberi suara untuk blok, dan blok jadi final
setelah didukung lebih dari 2/3 validator, tanpa menunggu epoch (mekanisme dari pengetahuan
umum). Hasilnya jauh lebih cepat: saat dicek, blok `finalized` cuma 4 blok di belakang
`latest`.

## Bukti (2026-10-10)

Data dari Beacon API dan JSON-RPC publicnode (Ethereum), dan `bsc-dataseed.bnbchain.org` (BSC).

| Cek | Hasil |
| --- | --- |
| `finality_checkpoints` saat head di epoch 481.228 (slot ke-1 epoch itu) | current justified = epoch **481.227**, finalized = epoch **481.226** |
| Root checkpoint vs root blok | justified 481.227 = `0xbb40…` = blok slot 15.399.264; finalized 481.226 = `0x92e3…` = blok slot 15.399.232 → keduanya blok pertama epoch-nya |
| Satu epoch sebelumnya (state slot 15.399.265, epoch 481.227) | justified 481.226, finalized 481.225 → keduanya maju tepat 1 setelah melewati batas epoch |
| Blok execution di dalam checkpoint finalized | 26.160.329 (`0xd165…`) = **sama persis** dengan `eth_getBlockByNumber("finalized")` |
| Blok execution di dalam checkpoint justified | 26.160.361 = **sama persis** dengan `eth_getBlockByNumber("safe")` |
| `latest` saat itu | 26.160.394 → `finalized` tertinggal **65 blok** (±13 menit) |
| BSC: `finalized` / `safe` / `latest` | 126.784.037 / 126.784.039 / 126.784.041 → `finalized` tertinggal **4 blok** (±1,8 detik dengan blok 0,45 detik) |
| State lebih lama dari ±1 epoch di Beacon API publik | HTTP 403, jadi riwayat checkpoint lebih panjang tidak bisa dibaca |

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)
