# 🧪 Module 2: First Real Program (Learning by Building)

**Goal:** Ship a smart contract exploring real-world patterns:

- client-side signing
- PDA-based authorization via `invoke_signed`
- mutable account updates
- CPI to SPL Token program
- on-chain logs for debugging

---

## 🔧 Project: Token Escrow Program

**Use Case:**

Alice locks SPL tokens into a PDA escrow vault so Bob can claim them upon action (e.g. sending stablecoins). Alice can also cancel and reclaim. This reflects common patterns in decentralized swaps, auctions, or escrow apps.

---

## ⚙️ Instructions Overview

❇ **1. `initialize_escrow`**

- Accounts:
  - Alice (signer), her token ATA, escrow state PDA, escrow vault PDA
  - SPL Token, System, Rent programs
- Logic:
  - Create and fund escrow state PDA with bump
  - Initialize vault ATA with PDA as authority
  - Transfer tokens from Alice’s ATA → vault ATA (via CPI)
  - Log with `msg!`

❇ **2. `complete_escrow`**

- Bob (signer) must use correct ATA; transfers and closes PDAs:
  - Transfer all tokens from vault ATA → Bob’s ATA (`invoke_signed`)
  - Close vault ATA and state PDA; rent refunded to initializer
  - Log each step

❇ **3. `cancel_escrow`**

- Alice reclaims funds:
  - Transfer tokens back from vault ATA → Alice’s ATA (`invoke_signed`)
  - Close vault ATA and state PDA
  - Log reclaim action

---

## 🧠 What You Learn

- ✅ **Client signer**: requires proper authority to execute
- ✅ **PDAs & `invoke_signed`**: vault ATA controlled securely by PDA
- ✅ **Mutable accounts**: state and vault accounts updated and closed
- ✅ **Cross-program invocation (CPI)**: interaction with SPL Token program
- ✅ **On-chain logs (`msg!`)**: essential for debugging and traceability

---

## 🧰 Tech Stack

- **Rust + Anchor v0.29.0**
- **`anchor_spl::token`** for CPI to SPL Token program
- **`invoke_signed`** to authorize PDAs for token transfers
- **TypeScript client** using `@solana/web3.js` to:
  - Derive PDAs
  - Build and send transactions
  - Confirm transactions and decode logs
- **Banks-client simulation** to mimic mainnet conditions locally

---

### 📚 References & Further Reading

- Anchor Escrow program example in *Anchor by Example* docs  [examples.anchor-lang.com](https://examples.anchor-lang.com/docs/non-custodial-escrow?utm_source=chatgpt.com) [HackMD](https://hackmd.io/%40ironaddicteddog/solana-anchor-escrow?utm_source=chatgpt.com) [GitHub](https://github.com/deanmlittle/anchor-escrow-2024?utm_source=chatgpt.com) [QuickNode](https://www.quicknode.com/guides/solana-development/anchor/how-to-write-your-first-anchor-program-in-solana-part-1?utm_source=chatgpt.com) [GitHub](https://github.com/ironaddicteddog/anchor-escrow?utm_source=chatgpt.com) [Solana Stack Exchange](https://solana.stackexchange.com/questions/7604/i-want-to-create-an-escrow-program-in-rust-on-solana?utm_source=chatgpt.com)
- Deep-dive tutorial from Block Magnates: “Write Your First Solana Escrow Contract with Anchor”  [Block Magnates](https://blog.blockmagnates.com/the-ultimate-guide-to-building-an-escrow-contract-on-solana-with-anchor-ceca1811bfd2?utm_source=chatgpt.com)
- CPI usage and PDAs: Anchor example docs  [anchor-lang.com](https://www.anchor-lang.com/docs/references/examples?utm_source=chatgpt.com)

---

Would you like me to write the actual code scaffolding next, including:

- Rust `lib.rs` with Anchor accounts and handlers?
- TypeScript script for initializing, completing, and cancelling escrows?
- Banks-client test suite simulating escrow flows with zero-cost tokens?

Let me know where you’d like to jump in!
