---
title: EVM 09.01 - Permit2
tags: [learning, evm, erc20, permit2, security]
updated: 2026-10-24
---

# Permit2

Bagian dari [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md)

Hasil percobaan untuk materi ini ada di [EVM 09.01 - Hasil Praktik](../Hasil%20Praktik/EVM%2009.01%20-%20Hasil%20Praktik.md).

**Permit2 adalah satu kontrak milik Uniswap yang memberi fitur "approve lewat tanda tangan" ke semua token ERC-20 — termasuk token yang tidak punya `permit` (EIP-2612) seperti USDT dan WETH.** Dirilis November 2022, alamatnya sama di semua chain EVM (deploy pakai CREATE2):

```
0x000000000022D473030F116dDEE9F6B43aC78BA3
```

## Masalah yang diselesaikan

Di [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md) ada dua cara memberi izin:

| Cara | Masalah |
| --- | --- |
| `approve` biasa | satu transaksi + gas **per token per dApp**; orang malas lalu approve unlimited; approval tidak pernah kedaluwarsa |
| `permit` (EIP-2612) | enak (tanda tangan saja), tapi **harus dibangun di dalam token**. Token lama (USDT, WETH) tidak punya dan tidak bisa diubah |

Permit2 mengambil jalan tengah: **approve satu kali ke Permit2** (unlimited, per token), lalu semua izin ke dApp diberikan lewat tanda tangan EIP-712 yang diverifikasi oleh Permit2. Permit2 yang memanggil `token.transferFrom(user, …)`, karena dialah yang memegang allowance on-chain.

```
User ──approve(Permit2, max)──▶ Token          (sekali per token, on-chain)
User ──tanda tangan EIP-712──▶ dApp            (off-chain, gratis)
dApp ──signature──▶ Permit2 ──transferFrom──▶ Token
```

## Dua mode

### 1. SignatureTransfer — sekali pakai

Tanda tangan berlaku untuk **satu transfer**, lalu mati. Tidak ada allowance yang tersisa. Ini yang dipakai Uniswap Universal Router dan UniswapX.

```solidity
struct TokenPermissions { address token; uint256 amount; }
struct PermitTransferFrom {
    TokenPermissions permitted;
    uint256 nonce;      // unordered: bitmap, bukan berurutan
    uint256 deadline;
}
struct SignatureTransferDetails { address to; uint256 requestedAmount; }

function permitTransferFrom(
    PermitTransferFrom calldata permit,
    SignatureTransferDetails calldata transferDetails,
    address owner,
    bytes calldata signature
) external;
```

- **`spender` tidak ada di struct**, tapi ikut di-hash: Permit2 memakai `msg.sender` sebagai spender saat memverifikasi. Jadi tanda tangan untuk router A tidak bisa dipakai kontrak B.
- **Nonce unordered** (bitmap): beberapa tanda tangan bisa dipakai dengan urutan bebas, cocok untuk order off-chain.
- **Witness** (`permitWitnessTransferFrom`): data tambahan (mis. detail order) ikut ditandatangani dalam satu tanda tangan. Inilah dasar UniswapX.

### 2. AllowanceTransfer — izin dengan batas waktu

Mirip `approve` biasa, tapi disimpan di Permit2 dan **punya kedaluwarsa**.

```solidity
struct PermitDetails {
    address token;
    uint160 amount;
    uint48  expiration;   // allowance mati otomatis setelah ini
    uint48  nonce;        // berurutan
}
struct PermitSingle {
    PermitDetails details;
    address spender;
    uint256 sigDeadline;  // batas waktu tanda tangan dipakai
}

function permit(address owner, PermitSingle calldata permitSingle, bytes calldata signature) external;
function transferFrom(address from, address to, uint160 amount, address token) external;
function lockdown(TokenSpenderPair[] calldata approvals) external;  // revoke massal
```

Ada dua deadline yang berbeda: `sigDeadline` (tanda tangan basi) dan `expiration` (allowance basi).

## Contoh: deposit USDT ke vault tanpa approve ke vault

Vault yang menerima deposit lewat SignatureTransfer:

```solidity
import {ISignatureTransfer} from "permit2/src/interfaces/ISignatureTransfer.sol";

contract Vault {
    ISignatureTransfer constant PERMIT2 =
        ISignatureTransfer(0x000000000022D473030F116dDEE9F6B43aC78BA3);
    mapping(address => mapping(address => uint256)) public balanceOf; // user => token => amount

    function depositWithPermit2(
        address token,
        uint256 amount,
        uint256 nonce,
        uint256 deadline,
        bytes calldata signature
    ) external {
        PERMIT2.permitTransferFrom(
            ISignatureTransfer.PermitTransferFrom({
                permitted: ISignatureTransfer.TokenPermissions({token: token, amount: amount}),
                nonce: nonce,
                deadline: deadline
            }),
            ISignatureTransfer.SignatureTransferDetails({to: address(this), requestedAmount: amount}),
            msg.sender,   // owner = pemanggil; tanda tangan harus dari dia
            signature
        );
        balanceOf[msg.sender][token] += amount;
    }
}
```

Alur dari sisi user:

1. **Sekali seumur hidup per token:** `USDT.approve(Permit2, max)` — satu transaksi.
2. Di dApp, wallet menampilkan **"Sign typed data"** `PermitTransferFrom` berisi token USDT, amount 100, spender = Vault, nonce, deadline.
3. User memanggil `depositWithPermit2(...)` dengan tanda tangan itu → Permit2 cek tanda tangan → `USDT.transferFrom(user, vault, 100)`.

Deposit ke vault lain berikutnya: **tidak perlu approve lagi**, cukup tanda tangan baru. Tanda tangan yang sama tidak bisa diulang (nonce terpakai) dan tidak bisa dipakai kontrak lain (spender = `msg.sender`).

Domain EIP-712-nya: `name = "Permit2"`, `chainId`, `verifyingContract = 0x0000…8BA3` (tanpa `version`). Rumus digest sama dengan permit di EVM 09: `keccak256("\x19\x01" ‖ DOMAIN_SEPARATOR ‖ structHash)`.

## Sisi gelap: phishing jadi lebih kuat

Karena hampir semua orang yang pernah swap di Uniswap sudah approve unlimited ke Permit2, **satu tanda tangan Permit2 bisa menguras token apa pun** yang sudah di-approve ke Permit2 — bahkan beberapa token sekaligus lewat `PermitBatch`. Sama seperti skenario 3 di EVM 09: korban tidak mengirim transaksi, tidak bayar gas, yang terlihat cuma "Sign typed data".

Yang harus dibaca di setiap tanda tangan dengan `verifyingContract` = Permit2:

| Field | Pertanyaan |
| --- | --- |
| `spender` | ini alamat dApp yang aku kenal? |
| `token` / `amount` | sesuai yang mau aku lakukan, bukan `max`? |
| `expiration` / `deadline` | wajar (menit/jam), bukan bertahun-tahun? |

Revoke: `lockdown` di Permit2 untuk allowance ke spender tertentu; atau revoke approval token → Permit2 (revoke.cash menampilkan keduanya).

## Jebakan: alamat test terkenal bisa sudah punya code

Permit2 memeriksa `owner.code.length`. Kalau lebih dari 0, Permit2 memperlakukan owner sebagai
kontrak dan memverifikasi lewat **EIP-1271** (`isValidSignature`), bukan `ecrecover`. Tanda tangan
ECDSA biasa dari owner itu jadi ditolak.

Sejak EIP-7702, EOA pun bisa punya code: `0xef0100` + alamat kontrak tujuan (lihat
[EVM 02 - Account Model](../EVM%2002%20-%20Account%20Model.md)). Private key test yang terkenal — misalnya `0xA11CE`, atau
`makeAddrAndKey("alice")` dari forge-std — sudah publik, jadi siapa pun (biasanya bot sweeper) bisa
memasang delegasi di alamat itu di mainnet. Test di fork mainnet yang memakai key tersebut bisa gagal
dengan revert di `isValidSignature`.

- Di fork mainnet, **jangan pakai key test terkenal**. Pakai label unik, dan cek
  `assertEq(alice.code.length, 0)` di `setUp`.
- Sejak EIP-7702, "alamat punya code" **tidak berarti** kontrak biasa. Kontrak yang bercabang ke
  EIP-1271 berdasarkan `code.length` (Permit2, OpenZeppelin `SignatureChecker`) ikut terpengaruh.

## Coba sendiri

Fork Ethereum supaya Permit2 dan USDT yang dipakai adalah kontrak asli.

1. Salin interface minimal Permit2 dari repo Uniswap, lalu tulis `Vault` seperti contoh di atas.
2. Di `setUp`, satu-satunya approve on-chain: `USDT.approve(Permit2, max)`. USDT tidak mengembalikan
   `bool`, tapi Permit2 memakai `SafeTransferLib`, jadi tetap jalan.
3. Deposit ke vault A hanya dengan tanda tangan, lalu cek `USDT.allowance(alice, vault)` tetap 0.
   Deposit ke vault B dengan tanda tangan baru, tanpa approve baru.
4. Coba nonce besar dulu lalu nonce kecil (unordered nonce), pakai ulang tanda tangan, pakai
   tanda tangan vault A di vault B, dan lewati deadline.
5. Simulasikan phishing `PermitSingle` max ke attacker, lalu coba `expiration` dan `lockdown`.

```bash
forge test --fork-url https://eth.drpc.org -vv
cast code <alamat>                     # cek apakah alamat test punya code (delegasi EIP-7702)
```

## Ringkas

| | `approve` | EIP-2612 `permit` | Permit2 |
| --- | --- | --- | --- |
| Butuh dukungan token | tidak | **ya** | tidak |
| Transaksi per dApp | 1 | 0 | 0 (setelah approve sekali ke Permit2) |
| Kedaluwarsa | tidak | tidak (allowance) | ya (`expiration`) / sekali pakai |
| Risiko phishing tanda tangan | — | per token | **semua token yang di-approve ke Permit2** |

Referensi: repo `Uniswap/permit2` di GitHub, docs di docs.uniswap.org (bagian Permit2).

---

Bagian dari [EVM 09 - Bahaya Approval ERC-20](../EVM%2009%20-%20Bahaya%20Approval%20ERC-20.md)
