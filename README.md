# 🚀 Solana Smart Contract Development — The No-Bullshit, Bottom-Up Guide

This course is designed for engineers who learn best by **doing**. No fluff. No unnecessary theory. We get our hands dirty, understand what’s under the hood, and build up from there.

---

## [🔧 Module 1: Environment Setup (Just Do It™)](module-1)

Before writing any line of code, we set up the dev environment like real engineers.

- ✅ Install Rust (stable toolchain)
- ✅ Install Solana CLI
- ✅ Install Node.js (for web3.js and client development)
- ✅ Install Anchor **v0.29.0** — not 0.30.0, because stability > bleeding edge
- ✅ Create `solana-test-validator` with custom config

We skip the history lessons and focus on setting up a repeatable local environment that works.

---

## 🧪 [Module 2: First Real Program (Learning by Building)](module-2)

**Goal:** Build a real smart contract that touches the core building blocks of Solana.

### ✨ Features

- ✅ Instruction requires **signer from the client**
- ✅ Instruction triggers **invoke_signed** to prove PDA ownership
- ✅ Instruction writes to **mutable accounts**

This will teach:

- Account signing from client
- PDA authority and signed invocation (`invoke_signed`)
- Account mutability
- On-chain logs and debugging

---

## 🌐 Module 3: The No-BS Web3 — Client Calls That Actually Work

We write the frontend and scripts **right after** the program is done.

### Topics

- ✅ Generate a PDA (on the client)
- ✅ Send real instructions using `@solana/web3.js`
- ✅ Understand instruction data layout and encoding
- ✅ Call instructions **manually** (no Anchor client helper)
- ✅ Use Anchor's IDL if desired, but know how to go without it

We call our own program using raw transactions.

---

## 🧪 Module 4: Simulating Mainnet With Banks Client

We want real-world realism **without** spending SOL.

### Tools

- ✅ Anchor's `banks-client` in TypeScript
- ✅ Import real devnet/mainnet accounts locally
- ✅ Load historical states to simulate live interactions
- ✅ Run tests with realistic setup: signer, PDA, and mutable states

This approach builds real muscle memory for production-grade testing.

---

## 🔍 Module 5: How Anchor Really Works (Not Magic)

We pause to understand what's happening under the hood.

### Key Concepts

- **Account discriminators:** 8-byte prefixes to identify accounts
- **Instruction discriminators:** `global:<ix_name>` in snake_case
- Anchor macros like `#[derive(Accounts)]`, `#[account(mut)]`, `#[instruction(args...)]`
- Where your program data really lives (`account.data.borrow_mut()`)

This module is where Anchor goes from being a black box to being transparent and predictable.

---

## 🛠️ Module 6: Rewriting It Without Anchor

Now that learners understand what Anchor does, we strip it away.

### Raw Solana Program

- No macros
- Manual deserialization
- Manual PDA validation
- `invoke_signed` by hand
- Borsh or raw byte-level work

This is where we strengthen the foundation. Anchor becomes optional, not essential.

---

## 🧱 Module 7: Back to Anchor (Now With Context)

Now that learners appreciate the abstraction, we go back and:

- Refactor the raw program into Anchor again
- Compare LOC, developer experience, and security safety

This helps learners understand when to use Anchor — and when not to.

---

## 🧠 Module 8: Higher-Level Concepts & Solana Vision

With hands-on skills built, we now explore:

- What Solana is optimized for
- History of Sealevel execution
- Parallelism and why it's fast
- Real-world use cases (DeFi, NFTs, identity, games)

This gives learners a **vision** of what they can build — and why Solana matters.

---

## 📎 Extra: Solana Program Security (Sealevel Gotchas)

- Solana’s parallel execution model
- Account constraints and limitations
- CPI safety
- Reentrancy? Not a problem if you structure your program right.
- Anchor account constraints vs manual validation

---

## 🧰 Tools We Use

- `solana-test-validator`
- `solana logs`
- `anchor test`
- `solana program dump`
- `@solana/web3.js`
- `banks-client` (TypeScript)

---

## ✅ Prerequisites

- Knows basic JavaScript/TypeScript
- Comfortable with CLI and editing code
- Has 0 knowledge of Solana? Perfect.
- **Node.js** (for web3.js and client-side development)

---

## 🎓 Outcome

By the end of this course, you will:

- Write Solana programs with and without Anchor
- Understand signing, PDAs, mutability, and serialization
- Build real-world apps that interact with your contracts
- Know how to debug, test, and secure your programs
- Think like a Solana engineer, not just a copy-paster

---

## 📝 Credits

This guide was written by **ChatGPT**, with the course structure and bottom-up philosophy guided by **[ghabxph](https://github.com/ghabxph)**.

*Solana is not magic. Let's prove it.*
