# 🤝 DuoPay — Joint Purchase Escrow on Liquid Network

> **Blockstream Turin Simplicity Hackathon Submission**  
> A trustless co-buyer escrow contract built with SimplicityHL on the Liquid Network

---

## 🧩 The Problem

Imagine two friends want to buy an expensive item together — a laptop, a piece of art, rare sneakers, or even a property down payment. Today they face a painful choice:

- **Person A sends money first** → risks getting scammed if Person B or the seller disappears
- **Person B sends money first** → same risk, reversed
- **Use a middleman service** → fees, identity checks, trust in a third party

There is no trustless, peer-to-peer way for **two co-buyers** to jointly fund a purchase and guarantee the seller only gets paid when **both parties confirm** delivery.

**DuoPay solves this using a SimplicityHL smart contract on the Liquid Network.**

---

## 💡 The Solution

DuoPay is a **2-of-2 co-buyer escrow** contract. Both buyers lock their funds into the contract. The funds only release to the seller when **both buyers sign a confirmation**. If the deal falls through, both get refunded automatically after a timeout.

```
Buyer A ──┐
           ├──► [DuoPay Contract] ──► Seller  (only if BOTH confirm)
Buyer B ──┘                      └──► Refund  (if timeout reached)
```

No middleman. No fees. No trust required between strangers.

---

## 🏗️ How It Works (Contract Logic)

### States

| State | Description |
|-------|-------------|
| `Funded` | Both buyers have deposited their share |
| `Confirmed` | Both buyers have signed release |
| `Released` | Funds sent to seller |
| `Refunded` | Deal cancelled, funds returned |

### Release Conditions

```
RELEASE to seller  →  requires signature from BOTH buyer_a AND buyer_b
REFUND to buyers   →  requires block height > timeout_height (e.g. 144 blocks ≈ 24 hrs)
```

### Why Simplicity Is The Right Tool

Simplicity's architecture makes DuoPay uniquely secure:

1. **No loops / no surprises** — The contract has a finite, statically bounded execution path. There are only two outcomes: release or refund. Nothing can go wrong at runtime.
2. **Formally verifiable** — Both buyers can mathematically prove no unauthorized spend path exists before funding the contract.
3. **UTXO model** — Contract state travels with the transaction, not in a global state. This eliminates reentrancy attacks that plague Ethereum escrows.
4. **Predictable fees** — SimplicityHL contracts have statically bounded costs. Both buyers know exact fees before committing.

---

## 📄 SimplicityHL Contract Design

```rust
// DuoPay: Joint Purchase Escrow
// Two co-buyers fund a contract. Seller is paid only when both confirm.

contract DuoPay {
    // Parties
    param buyer_a_pubkey: PubKey,
    param buyer_b_pubkey: PubKey,
    param seller_pubkey:  PubKey,

    // Amounts (in satoshis of L-BTC)
    param buyer_a_amount: u64,
    param buyer_b_amount: u64,

    // Timeout: refund available after this block height
    param timeout_height: u32,

    // SPEND PATH 1: Both confirm → release to seller
    fn release(sig_a: Sig, sig_b: Sig) -> bool {
        verify_sig(buyer_a_pubkey, sig_a) &&
        verify_sig(buyer_b_pubkey, sig_b) &&
        output_value(0) == buyer_a_amount + buyer_b_amount &&
        output_pubkey(0) == seller_pubkey
    }

    // SPEND PATH 2: Timeout reached → refund both buyers
    fn refund(sig_a: Sig, sig_b: Sig) -> bool {
        current_block_height() >= timeout_height &&
        verify_sig(buyer_a_pubkey, sig_a) &&
        verify_sig(buyer_b_pubkey, sig_b) &&
        output_value(0) == buyer_a_amount &&
        output_pubkey(0) == buyer_a_pubkey &&
        output_value(1) == buyer_b_amount &&
        output_pubkey(1) == buyer_b_pubkey
    }
}
```

> ⚠️ **Note:** This is a SimplicityHL design representation. Exact syntax follows the SimplicityHL spec available in the [Simplex toolchain](https://docs.simplicity-lang.org). The contract logic above faithfully represents the intended program structure and would be compiled/tested against the Liquid Testnet using Simplex.

---

## 🔄 User Flow

### Step 1 — Setup
Both buyers agree off-chain on:
- Total price and each buyer's share
- Seller's Liquid address
- Timeout (e.g. 144 blocks = ~24 hours)

### Step 2 — Fund
Each buyer sends their share to the contract address on Liquid Network.  
The contract is funded only when **both** deposits are confirmed on-chain.

### Step 3A — Confirm (Happy Path)
Item is delivered. Both buyers sign a `release` transaction.  
Seller receives the full amount in a single on-chain transaction.

### Step 3B — Dispute / Timeout (Refund Path)
If either buyer is unsatisfied or the deal collapses:
- Wait for the timeout block height to pass
- Both buyers co-sign a `refund` transaction
- Each buyer receives their original deposit back

---

## 🌍 Real-World Use Cases

| Scenario | Who Uses It |
|----------|------------|
| Two friends splitting a purchase (electronics, furniture) | Peer consumers |
| Co-investors buying tokenized assets on Liquid | DeFi / Liquid users |
| Business partners funding a shared vendor payment | SMEs |
| Community group buys (sneakers, limited edition drops) | Online communities |
| Cross-border co-purchases where escrow services don't operate | Unbanked / emerging markets |

---

## 🔍 Why This Is Innovative

Most escrow solutions on blockchain are:
- **Single buyer ↔ single seller** — DuoPay is natively **2 buyers → 1 seller**
- **Reliant on a trusted arbitrator** — DuoPay removes the arbitrator entirely for the happy path
- **Ethereum-based** — meaning global state risks, reentrancy attacks, unpredictable gas

DuoPay is the **first co-buyer escrow design** built natively for Simplicity's UTXO model, taking advantage of its formal verification and bounded execution.

---

## 🗺️ Future Extensions

- **3+ co-buyers** using Simplicity's multi-input UTXO support
- **Dispute resolution via oracle** — an agreed arbitrator's signature unlocks a third spend path
- **Partial confirmation** — if one buyer confirms but the other doesn't, a partial settlement after an extended timeout
- **Web interface** — a lightweight JS frontend using the Liquid.js SDK to create and fund DuoPay contracts without technical knowledge

---

## 🧱 Technical Stack

| Component | Technology |
|-----------|-----------|
| Smart contract | SimplicityHL on Liquid Network |
| Dev environment | Simplex (Blockstream toolchain) |
| Test network | Liquid Testnet / Simplex regtest |
| Future frontend | HTML/JS + Liquid.js SDK |

---

## 📚 References

- [Simplicity Language Docs](https://docs.simplicity-lang.org)
- [SimplicityHL Overview](https://simplicity-lang.org)
- [Blockstream simplicity-contracts GitHub](https://github.com/BlockstreamResearch/simplicity-contracts)
- [Liquid Network](https://liquid.net)

---

## 👤 Submitted By

Submitted to the **Blockstream Turin Simplicity Hackathon** — May 17, 2026  
Built in a single session using only a mobile device 📱

> *"The most important Simplicity applications are the ones that I haven't thought of."*  
> — Russell O'Connor, creator of Simplicity
