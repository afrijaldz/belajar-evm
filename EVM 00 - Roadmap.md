---
title: EVM 00 - Roadmap
tags: [learning, evm, roadmap]
updated: 2026-10-19
---

# EVM Roadmap

Materi belajar Ethereum Virtual Machine (EVM) dari dasar sampai bisa menulis, menguji, dan men-deploy smart contract. Setiap sesi membuktikan konsepnya lewat praktik: `anvil` lokal dan fork chain EVM sungguhan (terutama Ethereum dan BSC; Base dan Arbitrum di Sesi 23), dengan perintah dan hasilnya ditulis apa adanya.

Tidak ada prasyarat selain bisa menjalankan perintah di terminal. Mesin EVM sama di semua chain (lihat [EVM 01 - EVM vs Non-EVM](EVM%2001%20-%20EVM%20vs%20Non-EVM.md)), jadi semua materi berlaku lintas chain EVM.

Tools: [Foundry](https://book.getfoundry.sh/) (`forge`, `cast`, `anvil`).

## Cara membaca

Baca **berurutan dari nomor terkecil**. Semua catatan diberi awalan `EVM NN - …` sehingga sudah tersusun sesuai urutan. Tiap catatan berdiri sendiri tapi sering menaut ke nomor sebelumnya — kalau ada istilah asing, ikuti tautannya mundur.

Beberapa sesi punya folder `EVM NN - Detail/` berisi penjelasan mendalam per subtopik. Token nyata seperti UNI, CAKE, dan USDT dipakai sebagai contoh supaya konsepnya terlihat di kondisi sesungguhnya, bukan cuma di contoh buatan.

## Urutan baca

| No | Catatan | Isi |
| ---: | --- | --- |
| **Fase 1** | **Dasar EVM** | |
| 01 | [EVM 01 - EVM vs Non-EVM](EVM%2001%20-%20EVM%20vs%20Non-EVM.md) | mesin vs native token, BEP-20 = ERC-20, HyperCore vs HyperEVM; detail per komponen (opcode, stack, memory, storage, PC, gas, alamat, RPC, native token, EVM vs node vs validator, execution vs consensus client, validator client, slashing, attestation, finality) di folder `EVM 01 - Detail/` (`EVM 01.01`–`01.18`) |
| 02 | [EVM 02 - Account Model](EVM%2002%20-%20Account%20Model.md) | empat field akun, alamat kontrak, storage mentah, `0xdead`, EIP-7702; detail di folder `EVM 02 - Detail/` (`EVM 02.01`–`02.02`) |
| 03 | [EVM 03 - Transaksi dan Gas](EVM%2003%20-%20Transaksi%20dan%20Gas.md) | anatomi transaksi, EIP-1559, rumus base fee, harga calldata/storage, log |
| 04 | [EVM 04 - ABI](EVM%2004%20-%20ABI.md) | selector, encoding, event, decode transaksi burn CAKE |
| 05 | [EVM 05 - Fork dan Foundry](EVM%2005%20-%20Fork%20dan%20Foundry.md) | fork BSC, archive node, impersonate, `setStorageAt`, `forge test` di fork |
| 06 | [EVM 06 - Mekanisme Burn UNI di Fork](EVM%2006%20-%20Mekanisme%20Burn%20UNI%20di%20Fork.md) | bot searcher, Firepit + TokenJar, burn UNI sendiri, aturan mint 2% |
| **Fase 2** | **Solidity dan Token** | |
| 07 | [EVM 07 - Dasar Solidity](EVM%2007%20-%20Dasar%20Solidity.md) | overflow, packing, storage/memory/calldata, visibility, modifier, custom error, `payable`/`receive()`; detail di folder `EVM 07 - Detail/` (`EVM 07.01`–`07.02`) |
| 08 | [EVM 08 - ERC-20 dari Nol](EVM%2008%20-%20ERC-20%20dari%20Nol.md) | token dari nol, fuzz test, bug self-transfer; bacaan lanjutan: B20 Token Standard Base (ERC-20 + compliance di Base) |
| 09 | [EVM 09 - Bahaya Approval ERC-20](EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) | race approve, unlimited approve, phishing permit, USDT |
| 10 | [EVM 10 - Burn _burn vs 0xdead](EVM%2010%20-%20Burn%20_burn%20vs%200xdead.md) | dua gaya burn, supply efektif, peta pola burn |
| 11 | [EVM 11 - Events dan Indexing](EVM%2011%20-%20Events%20dan%20Indexing.md) | indexer mini, filter topic, batas `eth_getLogs`, reorg |
| 12 | [EVM 12 - Mini Project Vault](EVM%2012%20-%20Mini%20Project%20Vault.md) | vault ETH + ERC-20, 3 jebakan, invariant + mutation test |
| **Fase 3** | **DeFi dan Integrasi** | |
| 13 | [EVM 13 - AMM x·y=k](EVM%2013%20-%20AMM%20x%C2%B7y%3Dk.md) | harga dari reserve, rumus swap, price impact, fee protokol; detail di folder `EVM 13 - Detail/` (`EVM 13.01`) |
| 14 | [EVM 14 - Swap dari Kontrak dan Sandwich](EVM%2014%20-%20Swap%20dari%20Kontrak%20dan%20Sandwich.md) | deadline, `amountOutMin`, simulasi sandwich, quote on-chain bukan proteksi |
| 15 | [EVM 15 - Uniswap v3 Likuiditas Terkonsentrasi](EVM%2015%20-%20Uniswap%20v3%20Likuiditas%20Terkonsentrasi.md) | `sqrtPriceX96` dan tick, v3 vs v2, posisi sempit/lebar/penuh, keluar rentang |
| 16 | [EVM 16 - Buyback-and-Burn Sendiri](EVM%2016%20-%20Buyback-and-Burn%20Sendiri.md) | desain keeper (CAKE) vs lelang (UNI), sandwich, kebocoran, threshold |
| 17 | [EVM 17 - Proxy dan Upgrade](EVM%2017%20-%20Proxy%20dan%20Upgrade.md) | `delegatecall`, tabrakan storage, EIP-1967, upgrade append-only, initializer, UUPS, Safe dan USDC; detail di folder `EVM 17 - Detail/` (`EVM 17.01`) |
| 18 | [EVM 18 - Security Basics](EVM%2018%20-%20Security%20Basics.md) | oracle spot vs TWAP + flash loan Morpho, `tx.origin`, **checklist auditor**; detail di folder `EVM 18 - Detail/` (`EVM 18.01` flash loan di mata auditor) |
| **Fase 4** | **Production** | |
| 19 | [EVM 19 - Testing Lanjutan](EVM%2019%20-%20Testing%20Lanjutan.md) | coverage cabang, differential vs router asli, ghost variable, shrinking, fork cache |
| 20 | [EVM 20 - Gas Optimization](EVM%2020%20-%20Gas%20Optimization.md) | baseline + benchmark transaksi nyata, optimizer, `encodeCall`, `unchecked`, trade-off storage |
| 21 | [EVM 21 - Deploy ke Testnet dan Verifikasi](EVM%2021%20-%20Deploy%20ke%20Testnet%20dan%20Verifikasi.md) | wallet deployer, RPC testnet, `forge script`, simulasi + fork, prinsip verifikasi |
| 22 | [EVM 22 - Monitoring](EVM%2022%20-%20Monitoring.md) | monitor event + solvency + reorg, laju burn UNI, burn L2 lewat bridge |
| 23 | [EVM 23 - Multi-Chain](EVM%2023%20-%20Multi-Chain.md) | 4 chain berdampingan, `block.number` Arbitrum, CREATE vs CREATE2, replay, invariant bridge UNI |
| 24 | [EVM 24 - Capstone SupplyWatch](EVM%2024%20-%20Capstone%20SupplyWatch.md) | kontrak supply efektif on-chain tanpa owner, fork UNI + CAKE, CREATE2 multi-chain |

## Isi tiap fase

### Fase 1: Dasar EVM (Sesi 1-6)

- Sesi 1: EVM vs non-EVM: mesin eksekusi vs native token, chain ID, BEP-20 = ERC-20
- Sesi 2: Model akun: EOA vs contract account, nonce, balance, code, storage
- Sesi 3: Transaksi dan gas: gas limit, base fee, priority fee, EIP-1559, receipt, log
- Sesi 4: ABI: function selector, encoding argumen, event topic (bongkar `0x70a08231` dan `Transfer`)
- Sesi 5: Setup Foundry + `anvil` lokal + fork mainnet/BSC
- Sesi 6: Baca kontrak nyata di fork: CAKE, UNI, MasterChef — dengan `cast` dan `forge`

### Fase 2: Solidity dan Token (Sesi 7-12)

- Sesi 7: Dasar Solidity: tipe data, storage vs memory, visibility, modifier
- Sesi 8: Tulis ERC-20 sendiri + test di Foundry
- Sesi 9: `approve` / `transferFrom` / allowance, dan kenapa approval bisa berbahaya
- Sesi 10: Burn: `_burn()` vs kirim ke `0xdead` — bandingkan `totalSupply`
- Sesi 11: Events dan indexing: `eth_getLogs`, topic filter, batas blok RPC
- Sesi 12: Mini project: vault deposit/withdraw ERC-20 + ETH

### Fase 3: DeFi dan Integrasi (Sesi 13-18)

- Sesi 13: AMM `x·y=k`: baca pool Uniswap v2 / PancakeSwap v2 di fork
- Sesi 14: Swap dari kontrak: router, slippage, deadline
- Sesi 15: Konsentrasi likuiditas: tick dan posisi di Uniswap v3
- Sesi 16: Buyback-and-burn dari kontrak: tulis versi sendiri
- Sesi 17: Proxy dan upgrade: Gnosis Safe, transparent/UUPS proxy, `masterCopy()`
- Sesi 18: Security basics: reentrancy, access control, oracle manipulation, audit mindset

### Fase 4: Production (Sesi 19-24)

- Sesi 19: Testing lanjutan: fuzz, invariant, fork test di Foundry
- Sesi 20: Gas optimization: storage slot, packing, `calldata`
- Sesi 21: Deploy ke testnet, verifikasi source di explorer
- Sesi 22: Monitoring: event listener, alert saldo/transfer
- Sesi 23: Multi-chain: deploy kontrak yang sama ke BSC, Base, Arbitrum
- Sesi 24: Capstone: kontrak kecil yang memantau burn/buyback token dan bisa dicek siapa pun

## Referensi

- Ethereum docs (EVM, akun, gas): https://ethereum.org/en/developers/docs/
- Solidity docs: https://docs.soliditylang.org/
- Foundry Book: https://book.getfoundry.sh/
- evm.codes (opcode dan gas): https://www.evm.codes/
- OpenZeppelin Contracts: https://docs.openzeppelin.com/contracts/
- BNB Chain docs: https://docs.bnbchain.org/
