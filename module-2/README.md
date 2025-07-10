# 🧪 Module 2: Anchor Project Setup (The Foundation)

> **🎯 Goal:** Create an Anchor project boilerplate for an escrow program - the "Hello World" of Solana development.

This module sets up the foundation for our escrow program that will teach all the core Solana concepts in Module 3.

---

## 📁 Create the Anchor Project

```bash
# Create a new Anchor project
anchor init escrow-program
cd escrow-program
```

This creates the standard Anchor project structure:

```text
escrow-program/
├── Anchor.toml          # Project configuration
├── Cargo.toml           # Rust dependencies
├── programs/            # Your Solana programs
│   └── escrow-program/
│       ├── Cargo.toml   # Program-specific dependencies
│       └── src/
│           └── lib.rs   # Main program logic
├── tests/               # Test files
│   └── escrow-program.ts
└── target/              # Build artifacts
```

---

## ⚙️ Configure Anchor.toml

Update your `Anchor.toml` for the escrow program:

```toml
[features]
seeds = false
skip-lint = false

[programs.localnet]
escrow_program = "Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS"

[registry]
url = "https://api.apr.dev"

[provider]
cluster = "localnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
```

**Key points:**

- `escrow_program` is your program ID (Anchor generates this)
- `cluster = "localnet"` for local development
- `wallet` points to your Solana keypair

---

## 🏗️ Basic Program Skeleton

Replace `programs/escrow-program/src/lib.rs` with:

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod escrow_program {
    use super::*;

    // Initialize escrow
    pub fn initialize_escrow(
        ctx: Context<InitializeEscrow>,
        amount: u64,
    ) -> Result<()> {
        msg!("Initializing escrow with {} lamports", amount);
        Ok(())
    }

    // Complete escrow
    pub fn complete_escrow(ctx: Context<CompleteEscrow>) -> Result<()> {
        msg!("Completing escrow");
        Ok(())
    }

    // Cancel escrow
    pub fn cancel_escrow(ctx: Context<CancelEscrow>) -> Result<()> {
        msg!("Canceling escrow");
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeEscrow<'info> {
    #[account(mut)]
    pub escrow_account: Account<'info, EscrowAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct CompleteEscrow<'info> {
    #[account(mut)]
    pub escrow_account: Account<'info, EscrowAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct CancelEscrow<'info> {
    #[account(mut)]
    pub escrow_account: Account<'info, EscrowAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct EscrowAccount {
    pub amount: u64,
    pub owner: Pubkey,
    pub is_active: bool,
}

impl EscrowAccount {
    pub const LEN: usize = 8 + 8 + 32 + 1; // discriminator + amount + owner + is_active
}
```

**What this skeleton includes:**

- ✅ Program ID declaration
- ✅ Three main instructions (initialize, complete, cancel)
- ✅ Account structures for each instruction
- ✅ Escrow account data structure
- ✅ Basic logging with `msg!()`

---

## 🧪 Test Setup

Update `tests/escrow-program.ts`:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { EscrowProgram } from "../target/types/escrow_program";
import { PublicKey, Keypair, LAMPORTS_PER_SOL } from "@solana/web3.js";
import { expect } from "chai";

describe("escrow-program", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.EscrowProgram as Program<EscrowProgram>;

  // Test accounts
  const user = Keypair.generate();
  const escrowAccount = Keypair.generate();

  before(async () => {
    // Airdrop SOL to user
    const signature = await provider.connection.requestAirdrop(
      user.publicKey,
      2 * LAMPORTS_PER_SOL
    );
    await provider.connection.confirmTransaction(signature);
  });

  it("Can initialize escrow", async () => {
    const amount = new anchor.BN(1 * LAMPORTS_PER_SOL);

    await program.methods
      .initializeEscrow(amount)
      .accounts({
        escrowAccount: escrowAccount.publicKey,
        user: user.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([user, escrowAccount])
      .rpc();

    // Verify escrow account was created
    const escrowData = await program.account.escrowAccount.fetch(
      escrowAccount.publicKey
    );
    expect(escrowData.amount.toNumber()).to.equal(amount.toNumber());
    expect(escrowData.owner.toString()).to.equal(user.publicKey.toString());
    expect(escrowData.isActive).to.be.true;
  });
});
```

---

## 🚀 Build and Test Workflow

```bash
# Build the program
anchor build

# Run tests
anchor test

# Deploy to localnet (if you have a validator running)
anchor deploy
```

**Expected test output:**

```text
escrow-program
  ✓ Can initialize escrow (1234ms)

1 passing (2s)
```

---

## 📦 Install Dependencies

```bash
# Install Node.js dependencies
npm install

# Or if using yarn
yarn install
```

---

## ✅ Verify Everything Works

```bash
# Check Anchor version
anchor --version

# Build without errors
anchor build

# Tests pass
anchor test
```

---

## 🎯 What's Next?

Your Anchor project is now ready! In [Module 3](../module-3/README.md), we'll:

1. **Implement the actual escrow logic** with real account interactions
2. **Add PDA (Program Derived Address) handling** for secure escrow accounts
3. **Implement proper error handling** and validation
4. **Add comprehensive tests** for all escrow scenarios

**Pro tip:** Keep this boilerplate handy - you'll use this same structure for future Solana programs.

---

## 🔧 Troubleshooting

**Build errors?**

- Check Rust toolchain: `rustc --version`
- Verify Anchor version: `anchor --version`
- Clean and rebuild: `anchor clean && anchor build`

**Test failures?**

- Ensure you have a Solana keypair: `solana-keygen new`
- Check network connection for airdrops
- Verify account sizes match your data structures

**Deploy issues?**

- Make sure you have SOL in your wallet
- Check your cluster configuration in `Anchor.toml`
- Verify program ID matches in `lib.rs` and `Anchor.toml`
