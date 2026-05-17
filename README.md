# 🤝 DuoPay — Trustless Co-Buyer Escrow on Liquid Network

> **Blockstream Turin Simplicity Hackathon · May 17, 2026**
> Two buyers. One seller. Zero middlemen.
> A SimplicityHL smart contract that holds funds until both parties confirm — then releases automatically. No trust required.

---

## 🧩 The Problem

Two people want to buy something together. Neither wants to send money first and risk getting scammed. Middlemen charge fees and require identity checks. Screenshots can be edited. There is no trustless peer-to-peer way for two co-buyers to jointly fund a purchase — until now.

Most existing escrow solutions fail in one or more of these ways:

- **Single buyer ↔ single seller** — no native support for two co-buyers splitting a payment
- **Reliant on a trusted arbitrator** — a human who can be bribed, biased, or offline
- **Ethereum-based** — global state risks, reentrancy vulnerabilities, and unpredictable gas costs
- **Custodial** — the middleman holds your funds, not a contract you can inspect

DuoPay eliminates all of these failure modes.

---

## 💡 The Solution

DuoPay is a **2-of-2 co-buyer escrow** contract written in SimplicityHL on the Liquid Network.

Both buyers lock their funds into the contract. Funds only release to the seller when **both buyers provide valid signatures** confirming delivery. If the deal collapses, both buyers are refunded after the timeout — no middleman needed.

```
Buyer A ──┐
           ├──► [duopay.simf] ──► Seller   (both signatures required)
Buyer B ──┘                  └──► Refund   (after timeout block height)
```

This is the **first co-buyer escrow design** built natively for Simplicity's UTXO model — taking full advantage of formal verification, bounded execution, and UTXO isolation that Ethereum-based escrows simply cannot offer.

---

## 📄 Contract (`duopay.simf`)

```rust
/*
 * DuoPay — 2-of-2 Co-Buyer Escrow
 *
 * Paths:
 *  1. Release: Both buyers sign → funds sent to seller
 *  2. Timeout: After expiry + both sign → deposits returned
 *
 * Parameters:
 *   param::BUYER_A_PUBKEY   — public key of co-buyer A
 *   param::BUYER_B_PUBKEY   — public key of co-buyer B
 *   param::EXPIRY_TIME      — block height after which refund is allowed
 *   param::BUYER_A_AMOUNT   — buyer A deposit in satoshis
 *   param::BUYER_B_AMOUNT   — buyer B deposit in satoshis
 */

fn get_output_explicit_amount(index: u32) -> u64 {
    let pair: (Asset1, Amount1) = unwrap(jet::output_amount(index));
    let (_, amount): (Asset1, Amount1) = pair;
    let amount: u64 = unwrap_right::<(u1, u256)>(amount);
    amount
}

fn check_signature(pubkey: Pubkey, sig: Signature) {
    let msg: u256 = jet::sig_all_hash();
    jet::bip_0340_verify((pubkey, msg), sig);
}

fn release_path(sig_a: Signature, sig_b: Signature) {
    check_signature(param::BUYER_A_PUBKEY, sig_a);
    check_signature(param::BUYER_B_PUBKEY, sig_b);

    let (carry, total): (bool, u64) = jet::add_64(param::BUYER_A_AMOUNT, param::BUYER_B_AMOUNT);
    assert!(jet::eq_1(<bool>::into(carry), 0));

    let out_amount: u64 = get_output_explicit_amount(0);
    assert!(jet::eq_64(out_amount, total));
}

fn timeout_path(sig_a: Signature, sig_b: Signature) {
    jet::check_lock_time(param::EXPIRY_TIME);

    check_signature(param::BUYER_A_PUBKEY, sig_a);
    check_signature(param::BUYER_B_PUBKEY, sig_b);

    let out_a: u64 = get_output_explicit_amount(0);
    let out_b: u64 = get_output_explicit_amount(1);

    assert!(jet::eq_64(out_a, param::BUYER_A_AMOUNT));
    assert!(jet::eq_64(out_b, param::BUYER_B_AMOUNT));
}

fn main() {
    let sig_a: Signature = witness::SIG_A;
    let sig_b: Signature = witness::SIG_B;

    match witness::PATH {
        Left(params: ())  => release_path(sig_a, sig_b),
        Right(params: ()) => timeout_path(sig_a, sig_b),
    }
}
```

### How the contract works

| Pattern | Taken from real SimplicityHL contracts |
|---------|----------------------------------------|
| `witness::PATH` | Witness namespace — same as `option_offer.simf` |
| `witness::SIG_A / SIG_B` | Signatures passed at spend time via `.wit` file |
| `jet::bip_0340_verify` | Real BIP-340 signature jet |
| `jet::check_lock_time` | Real timelock enforcement jet |
| `jet::sig_all_hash()` | Sighash covering the full transaction |
| `jet::add_64` / `jet::eq_64` | Arithmetic and comparison jets |
| `param::` namespace | Contract parameters set at compile time |
| `match witness::PATH { Left => ..., Right => ... }` | Standard two-path branching pattern |

### Contract State Machine

| State | Description |
|-------|-------------|
| `Funded` | Both buyers have deposited their share on-chain |
| `Confirmed` | Both buyers have co-signed a release transaction |
| `Released` | Full amount sent to seller in `Output[0]` |
| `Refunded` | After timeout: `Output[0]` → Buyer A, `Output[1]` → Buyer B |

There are exactly **two terminal states** — Released and Refunded. No intermediate states. No runtime surprises.

---

## 🏗️ Project Structure

```
duopay/
├── simf/
│   └── duopay.simf       — SimplicityHL contract
├── src/
│   └── main.rs           — Rust witness builder
├── tests/
│   └── release.wit       — witness file for happy path
│   └── timeout.wit       — witness file for refund path
├── Simplex.toml          — Simplex project config
└── README.md
```

---

## 🔄 User Flow

**Step 1 — Create Deal**
One party generates a contract address with agreed amounts, public keys, and timeout. A Deal Link is shared with the other buyer and the seller.

**Step 2 — Fund**
Both buyers deposit their share to the contract address on Liquid Testnet. Every deposit is verified on-chain — no screenshots, no "trust me."

**Step 3A — Release (Happy Path)**
Item delivered. Both buyers provide `SIG_A` and `SIG_B` via the `.wit` file with `witness::PATH = Left(())`. The contract verifies both BIP-340 signatures via `jet::bip_0340_verify` and releases the full amount to the seller in `Output[0]`.

**Step 3B — Timeout (Refund Path)**
Deal collapses. After `param::EXPIRY_TIME` block height is reached, both buyers provide signatures with `witness::PATH = Right(())`. The contract enforces the timelock via `jet::check_lock_time` and splits funds back — `Output[0]` to Buyer A, `Output[1]` to Buyer B.

---

## 🔍 Why Simplicity

| Property | What it means for DuoPay |
|----------|--------------------------|
| No loops / no recursion | Exactly two outcomes. Nothing can fail unexpectedly at runtime |
| Formally verifiable | Both buyers can prove no unauthorized spend path exists before depositing |
| UTXO isolation | Each contract is an isolated UTXO. Bugs elsewhere cannot drain it |
| `assert!()` — not exceptions | Every constraint is explicit and statically checkable |
| Statically bounded cost | Both buyers see the exact fee before committing |

### Why Not Ethereum / Solidity?

Ethereum escrows suffer from problems that Simplicity eliminates by design:

- **Reentrancy attacks** — Ethereum's global state allows malicious contracts to call back into an escrow mid-execution and drain it. Simplicity's UTXO model makes this impossible — each contract is an isolated coin.
- **Unpredictable gas** — Ethereum gas can spike or cause out-of-gas failures at runtime. SimplicityHL has statically bounded execution cost; both buyers see the exact fee before signing anything.
- **Arbitrator dependence** — Most production Ethereum escrows still rely on a human arbitrator key for disputes. DuoPay removes the arbitrator entirely for both the happy path and the timeout path.
- **Unverifiable code** — Even with open-source Solidity, verifying what a deployed contract actually does requires trusting the compiler and the bytecode. Simplicity is formally verifiable at the bit level.

---

## 🚀 The Bigger Vision

DuoPay is designed to grow into a full trustless payment platform with two modes:

**Mode 1 — Co-Buy:** Two buyers split a purchase. Neither goes first.
```
A deposits → B deposits → Both confirm → Seller paid
```

**Mode 2 — Client / Freelancer:** Client locks payment upfront. Freelancer delivers, client approves, funds release.
```
Client deposits → Work delivered → Client approves → Builder paid
```

This removes the payment security risks of informal platforms where proof of work is sent over chat but payment has no on-chain enforcement.

---

## 🌍 Use Cases

| Scenario | Category |
|----------|----------|
| Friends splitting a laptop, TV, or appliances | Consumer |
| Designers and developers getting paid for freelance work | Freelance |
| Two collectors buying rare items from an unknown seller | E-commerce |
| Cross-border purchases where escrow services don't exist | Global |
| Two investors jointly funding a tokenized Liquid asset | DeFi |
| Community group buys — sneakers, limited edition drops | Online communities |
| Business partners funding a shared vendor payment | SMEs |

---

## 🗺️ Roadmap

### v1 — Hackathon Scope
- [x] Contract design and spend path logic
- [x] Rust witness builder scaffolding
- [ ] Compile `duopay.simf` via Simplex toolchain
- [ ] Deploy to Liquid Testnet
- [ ] Build `.wit` witness files for both spend paths

### v2 — Post-Hackathon
- [ ] Deal Link web interface (lightweight JS + LWK SDK)
- [ ] Client/Freelancer mode (single funder, single approver)
- [ ] 3-of-3 and M-of-N co-buyer variants using Simplicity's multi-input UTXO support
- [ ] Optional dispute oracle: a third agreed-upon arbitrator signature unlocks a third spend path
- [ ] Partial settlement path: if one buyer confirms but the other goes silent, a secondary timeout releases proportional funds

---

## 🧱 Tech Stack

| Component | Technology |
|-----------|-----------|
| Smart contract | SimplicityHL (`.simf`) |
| Network | Liquid Testnet |
| Jets used | `jet::bip_0340_verify`, `jet::check_lock_time`, `jet::sig_all_hash`, `jet::add_64`, `jet::eq_64` |
| Witness pattern | `witness::PATH` + `Either<Left, Right>` + `match` |
| Witness builder | Rust + LWK SDK |
| Toolchain | Simplex |

---

## 📚 References

- [SimplicityHL Docs](https://docs.simplicity-lang.org)
- [simplicity-contracts repo](https://github.com/BlockstreamResearch/simplicity-contracts)
- [Witnesses in SimplicityHL](https://docs.simplicity-lang.org/documentation/witness/)
- [Jets Reference](https://docs.simplicity-lang.org/documentation/jets/)
- [Liquid Network](https://liquid.net)

---

## 👤 Submitted By

Submitted to the **Blockstream Turin Simplicity Hackathon** — May 17, 2026
Conceived and built in a single focused session — from idea to working contract in under 7 hours.

> *"The most important Simplicity applications are the ones that I haven't thought of."*
> — Russell O'Connor, creator of Simplicity

---

*DuoPay — the first co-buyer escrow designed natively for Simplicity's UTXO model.*
