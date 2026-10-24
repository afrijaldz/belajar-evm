---
title: EVM 01.18 - Finality
tags: [learning, evm, ethereum, bsc]
updated: 2026-10-24
---

# Finality

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)

Hasil percobaan untuk materi ini ada di [EVM 01.18 - Hasil Praktik](../Hasil%20Praktik/EVM%2001.18%20-%20Hasil%20Praktik.md).

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
`finalized` sebelum menganggap deposit masuk.

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
Dalam praktik, blok `finalized` tertinggal sekitar 60-an blok (±13 menit) dari `latest`.

Tag RPC dan checkpoint saling terhubung langsung: blok execution di dalam checkpoint finalized
adalah blok yang dikembalikan `eth_getBlockByNumber("finalized")`, dan blok di dalam checkpoint
justified adalah `eth_getBlockByNumber("safe")`. Root kedua checkpoint selalu root blok pertama
epoch-nya, dan keduanya maju tepat 1 epoch setiap kali melewati batas epoch.

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
kembali mencapai 2/3 dan finality jalan lagi.

Ethereum mainnet pernah kehilangan finality dua kali pada Mei 2023, masing-masing kurang dari
sekitar satu jam, karena masalah di beberapa consensus client. Blok tetap jalan selama
kejadian itu.

## BSC

BSC punya *fast finality* (BEP-126): validator memberi suara untuk blok, dan blok jadi final
setelah didukung lebih dari 2/3 validator, tanpa menunggu epoch. Hasilnya jauh lebih cepat:
blok `finalized` cuma beberapa blok di belakang `latest`, sekitar 2 detik.

## Coba sendiri

1. Bandingkan nomor blok `finalized`, `safe`, dan `latest` di Ethereum dan di BSC.
2. Ambil `finality_checkpoints` dari Beacon API, lalu cocokkan root checkpoint dengan root blok
   pertama epoch-nya.
3. Cari blok execution di dalam checkpoint finalized dan justified, lalu bandingkan dengan
   blok `finalized` dan `safe` dari JSON-RPC.

```bash
BEACON=https://ethereum-beacon-api.publicnode.com
cast block finalized --field number --rpc-url https://ethereum-rpc.publicnode.com
cast block safe --field number --rpc-url https://ethereum-rpc.publicnode.com
cast block latest --field number --rpc-url https://ethereum-rpc.publicnode.com
cast block finalized --field number --rpc-url https://bsc-dataseed.bnbchain.org
curl -s $BEACON/eth/v1/beacon/states/head/finality_checkpoints
curl -s $BEACON/eth/v2/beacon/blocks/<root> | jq '.data.message.body.execution_payload.block_number'
```

**Jebakan:** Beacon API publik biasanya menolak state yang lebih lama dari sekitar 1 epoch
(HTTP 403), jadi riwayat checkpoint yang panjang tidak bisa dibaca dari sana.

---

Bagian dari [EVM 01 - EVM vs Non-EVM](../EVM%2001%20-%20EVM%20vs%20Non-EVM.md) · ← [EVM 01.17 - Attestation](EVM%2001.17%20-%20Attestation.md)
