# 🧪 Module 3: First Real Program (Learning by Building)

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

## 🚀 Implementation

Let's build the token escrow program step-by-step.

### 📁 Project Structure

```bash
# Start from Module 2's escrow project or create new one
anchor init token-escrow
cd token-escrow
```

### 🦀 Rust Implementation (`programs/token-escrow/src/lib.rs`)

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Mint, Token, TokenAccount, Transfer, CloseAccount};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod token_escrow {
    use super::*;

    pub fn initialize_escrow(
        ctx: Context<InitializeEscrow>,
        amount: u64,
    ) -> Result<()> {
        msg!("Initializing escrow for {} tokens", amount);
        
        let escrow_state = &mut ctx.accounts.escrow_state;
        escrow_state.initializer = ctx.accounts.initializer.key();
        escrow_state.mint = ctx.accounts.mint.key();
        escrow_state.amount = amount;
        escrow_state.vault_authority_bump = ctx.bumps.vault_authority;
        
        msg!("Escrow state created with PDA: {}", escrow_state.key());

        // Transfer tokens from initializer to vault
        let cpi_accounts = Transfer {
            from: ctx.accounts.initializer_token_account.to_account_info(),
            to: ctx.accounts.vault_token_account.to_account_info(),
            authority: ctx.accounts.initializer.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        
        token::transfer(cpi_ctx, amount)?;
        msg!("Transferred {} tokens to escrow vault", amount);

        Ok(())
    }

    pub fn complete_escrow(ctx: Context<CompleteEscrow>) -> Result<()> {
        msg!("Completing escrow transaction");
        
        let escrow_state = &ctx.accounts.escrow_state;
        
        // Create signer seeds for PDA
        let seeds = &[
            b"vault-authority",
            escrow_state.key().as_ref(),
            &[escrow_state.vault_authority_bump],
        ];
        let signer = &[&seeds[..]];

        // Transfer tokens from vault to receiver using invoke_signed
        let cpi_accounts = Transfer {
            from: ctx.accounts.vault_token_account.to_account_info(),
            to: ctx.accounts.receiver_token_account.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer);
        
        token::transfer(cpi_ctx, escrow_state.amount)?;
        msg!("Transferred {} tokens to receiver", escrow_state.amount);

        // Close the vault token account
        let close_accounts = CloseAccount {
            account: ctx.accounts.vault_token_account.to_account_info(),
            destination: ctx.accounts.initializer.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        };
        let close_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            close_accounts,
            signer,
        );
        token::close_account(close_ctx)?;
        msg!("Closed vault token account, rent refunded to initializer");

        Ok(())
    }

    pub fn cancel_escrow(ctx: Context<CancelEscrow>) -> Result<()> {
        msg!("Canceling escrow, refunding to initializer");
        
        let escrow_state = &ctx.accounts.escrow_state;
        
        // Create signer seeds for PDA
        let seeds = &[
            b"vault-authority",
            escrow_state.key().as_ref(),
            &[escrow_state.vault_authority_bump],
        ];
        let signer = &[&seeds[..]];

        // Transfer tokens back to initializer
        let cpi_accounts = Transfer {
            from: ctx.accounts.vault_token_account.to_account_info(),
            to: ctx.accounts.initializer_token_account.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer);
        
        token::transfer(cpi_ctx, escrow_state.amount)?;
        msg!("Refunded {} tokens to initializer", escrow_state.amount);

        // Close the vault token account
        let close_accounts = CloseAccount {
            account: ctx.accounts.vault_token_account.to_account_info(),
            destination: ctx.accounts.initializer.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        };
        let close_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            close_accounts,
            signer,
        );
        token::close_account(close_ctx)?;
        msg!("Closed vault token account, rent refunded to initializer");

        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeEscrow<'info> {
    #[account(mut)]
    pub initializer: Signer<'info>,
    
    pub mint: Account<'info, Mint>,
    
    #[account(
        mut,
        constraint = initializer_token_account.mint == mint.key(),
        constraint = initializer_token_account.owner == initializer.key(),
    )]
    pub initializer_token_account: Account<'info, TokenAccount>,
    
    #[account(
        init,
        payer = initializer,
        space = EscrowState::LEN,
        seeds = [b"escrow", initializer.key().as_ref(), mint.key().as_ref()],
        bump,
    )]
    pub escrow_state: Account<'info, EscrowState>,
    
    /// CHECK: This is a PDA, we derive it in the client
    #[account(
        seeds = [b"vault-authority", escrow_state.key().as_ref()],
        bump,
    )]
    pub vault_authority: UncheckedAccount<'info>,
    
    #[account(
        init,
        payer = initializer,
        token::mint = mint,
        token::authority = vault_authority,
        seeds = [b"vault", escrow_state.key().as_ref()],
        bump,
    )]
    pub vault_token_account: Account<'info, TokenAccount>,
    
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}

#[derive(Accounts)]
pub struct CompleteEscrow<'info> {
    #[account(mut)]
    pub receiver: Signer<'info>,
    
    #[account(mut)]
    /// CHECK: We validate this is the initializer from escrow state
    pub initializer: UncheckedAccount<'info>,
    
    #[account(
        mut,
        close = initializer,
        seeds = [b"escrow", initializer.key().as_ref(), escrow_state.mint.as_ref()],
        bump,
    )]
    pub escrow_state: Account<'info, EscrowState>,
    
    /// CHECK: This is a PDA
    #[account(
        seeds = [b"vault-authority", escrow_state.key().as_ref()],
        bump = escrow_state.vault_authority_bump,
    )]
    pub vault_authority: UncheckedAccount<'info>,
    
    #[account(
        mut,
        seeds = [b"vault", escrow_state.key().as_ref()],
        bump,
    )]
    pub vault_token_account: Account<'info, TokenAccount>,
    
    #[account(
        mut,
        constraint = receiver_token_account.mint == escrow_state.mint,
        constraint = receiver_token_account.owner == receiver.key(),
    )]
    pub receiver_token_account: Account<'info, TokenAccount>,
    
    pub token_program: Program<'info, Token>,
}

#[derive(Accounts)]
pub struct CancelEscrow<'info> {
    #[account(mut)]
    pub initializer: Signer<'info>,
    
    #[account(
        mut,
        close = initializer,
        constraint = escrow_state.initializer == initializer.key(),
        seeds = [b"escrow", initializer.key().as_ref(), escrow_state.mint.as_ref()],
        bump,
    )]
    pub escrow_state: Account<'info, EscrowState>,
    
    /// CHECK: This is a PDA
    #[account(
        seeds = [b"vault-authority", escrow_state.key().as_ref()],
        bump = escrow_state.vault_authority_bump,
    )]
    pub vault_authority: UncheckedAccount<'info>,
    
    #[account(
        mut,
        seeds = [b"vault", escrow_state.key().as_ref()],
        bump,
    )]
    pub vault_token_account: Account<'info, TokenAccount>,
    
    #[account(
        mut,
        constraint = initializer_token_account.mint == escrow_state.mint,
        constraint = initializer_token_account.owner == initializer.key(),
    )]
    pub initializer_token_account: Account<'info, TokenAccount>,
    
    pub token_program: Program<'info, Token>,
}

#[account]
pub struct EscrowState {
    pub initializer: Pubkey,
    pub mint: Pubkey,
    pub amount: u64,
    pub vault_authority_bump: u8,
}

impl EscrowState {
    pub const LEN: usize = 8 + 32 + 32 + 8 + 1; // discriminator + pubkey + pubkey + u64 + u8
}

#[error_code]
pub enum EscrowError {
    #[msg("Invalid mint for token account")]
    InvalidMint,
    #[msg("Invalid owner for token account")]
    InvalidOwner,
    #[msg("Insufficient tokens in account")]
    InsufficientTokens,
}
```

### 🔧 Dependencies (`programs/token-escrow/Cargo.toml`)

```toml
[dependencies]
anchor-lang = "0.29.0"
anchor-spl = "0.29.0"
```

---

## 🧪 Key Learning Points

**1. Client Signing (`#[account(mut)] pub initializer: Signer<'info>`)**
- `Signer<'info>` enforces that the account must sign the transaction
- Required for authority validation

**2. PDA Authority (`invoke_signed`)**
- `vault_authority` PDA controls the vault token account
- `invoke_signed` allows PDA to sign CPI calls to SPL Token program
- Seeds must match exactly: `[b"vault-authority", escrow_state.key().as_ref()]`

**3. Mutable Accounts (`#[account(mut)]`)**
- Token accounts need `mut` for balance changes
- Escrow state needs `mut` for data updates
- `close = initializer` automatically closes account and refunds rent

**4. Cross-Program Invocation (CPI)**
- `token::transfer()` calls SPL Token program
- `CpiContext::new_with_signer()` for PDA-signed calls
- Proper account validation prevents unauthorized access

**5. On-chain Logging (`msg!`)**
- Essential for debugging program execution
- Shows up in `solana logs` output
- Helps trace program flow and state changes

---

## 🌐 TypeScript Client Example

### 📁 Setup (`tests/token-escrow.ts`)

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { TokenEscrow } from "../target/types/token_escrow";
import { 
  PublicKey, 
  Keypair, 
  LAMPORTS_PER_SOL,
  SystemProgram,
  SYSVAR_RENT_PUBKEY 
} from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  createMint,
  createAccount,
  mintTo,
  getAccount,
} from "@solana/spl-token";
import { expect } from "chai";

describe("token-escrow", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.TokenEscrow as Program<TokenEscrow>;

  // Test keypairs
  let initializer: Keypair;
  let receiver: Keypair;
  let mint: PublicKey;
  let initializerTokenAccount: PublicKey;
  let receiverTokenAccount: PublicKey;

  // PDAs
  let escrowState: PublicKey;
  let vaultAuthority: PublicKey;
  let vaultTokenAccount: PublicKey;

  const escrowAmount = new anchor.BN(1000 * 10**6); // 1000 tokens (6 decimals)

  before(async () => {
    // Create test accounts
    initializer = Keypair.generate();
    receiver = Keypair.generate();

    // Airdrop SOL
    await Promise.all([
      provider.connection.requestAirdrop(initializer.publicKey, 2 * LAMPORTS_PER_SOL),
      provider.connection.requestAirdrop(receiver.publicKey, 2 * LAMPORTS_PER_SOL),
    ]);

    // Create mint
    mint = await createMint(
      provider.connection,
      initializer,
      initializer.publicKey,
      null,
      6
    );

    // Create token accounts
    initializerTokenAccount = await createAccount(
      provider.connection,
      initializer,
      mint,
      initializer.publicKey
    );

    receiverTokenAccount = await createAccount(
      provider.connection,
      receiver,
      mint,
      receiver.publicKey
    );

    // Mint tokens to initializer
    await mintTo(
      provider.connection,
      initializer,
      mint,
      initializerTokenAccount,
      initializer,
      2000 * 10**6 // 2000 tokens
    );

    // Derive PDAs
    [escrowState] = PublicKey.findProgramAddressSync(
      [
        Buffer.from("escrow"),
        initializer.publicKey.toBuffer(),
        mint.toBuffer(),
      ],
      program.programId
    );

    [vaultAuthority] = PublicKey.findProgramAddressSync(
      [
        Buffer.from("vault-authority"),
        escrowState.toBuffer(),
      ],
      program.programId
    );

    [vaultTokenAccount] = PublicKey.findProgramAddressSync(
      [
        Buffer.from("vault"),
        escrowState.toBuffer(),
      ],
      program.programId
    );
  });

  it("Initializes escrow", async () => {
    console.log("🔄 Initializing escrow...");
    
    // Get initial balance
    const initializerAccountBefore = await getAccount(
      provider.connection,
      initializerTokenAccount
    );
    console.log(`Initial balance: ${initializerAccountBefore.amount}`);

    await program.methods
      .initializeEscrow(escrowAmount)
      .accounts({
        initializer: initializer.publicKey,
        mint: mint,
        initializerTokenAccount: initializerTokenAccount,
        escrowState: escrowState,
        vaultAuthority: vaultAuthority,
        vaultTokenAccount: vaultTokenAccount,
        systemProgram: SystemProgram.programId,
        tokenProgram: TOKEN_PROGRAM_ID,
        rent: SYSVAR_RENT_PUBKEY,
      })
      .signers([initializer])
      .rpc();

    // Verify escrow state
    const escrowAccount = await program.account.escrowState.fetch(escrowState);
    expect(escrowAccount.initializer.toString()).to.equal(initializer.publicKey.toString());
    expect(escrowAccount.amount.toString()).to.equal(escrowAmount.toString());
    expect(escrowAccount.mint.toString()).to.equal(mint.toString());

    // Verify tokens transferred to vault
    const vaultAccount = await getAccount(provider.connection, vaultTokenAccount);
    expect(vaultAccount.amount.toString()).to.equal(escrowAmount.toString());

    // Verify initializer balance reduced
    const initializerAccountAfter = await getAccount(
      provider.connection,
      initializerTokenAccount
    );
    expect(initializerAccountAfter.amount).to.equal(
      initializerAccountBefore.amount - BigInt(escrowAmount.toNumber())
    );

    console.log("✅ Escrow initialized successfully!");
  });

  it("Completes escrow", async () => {
    console.log("🔄 Completing escrow...");

    // Get receiver balance before
    const receiverAccountBefore = await getAccount(
      provider.connection,
      receiverTokenAccount
    );
    console.log(`Receiver balance before: ${receiverAccountBefore.amount}`);

    await program.methods
      .completeEscrow()
      .accounts({
        receiver: receiver.publicKey,
        initializer: initializer.publicKey,
        escrowState: escrowState,
        vaultAuthority: vaultAuthority,
        vaultTokenAccount: vaultTokenAccount,
        receiverTokenAccount: receiverTokenAccount,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .signers([receiver])
      .rpc();

    // Verify tokens transferred to receiver
    const receiverAccountAfter = await getAccount(
      provider.connection,
      receiverTokenAccount
    );
    expect(receiverAccountAfter.amount).to.equal(
      receiverAccountBefore.amount + BigInt(escrowAmount.toNumber())
    );

    // Verify escrow state is closed
    try {
      await program.account.escrowState.fetch(escrowState);
      expect.fail("Escrow state should be closed");
    } catch (error) {
      expect(error.message).to.include("Account does not exist");
    }

    console.log("✅ Escrow completed successfully!");
  });

  // Alternative test for canceling escrow
  it("Can cancel escrow", async () => {
    console.log("🔄 Testing cancel escrow...");

    // Initialize new escrow for cancel test
    const cancelAmount = new anchor.BN(500 * 10**6);

    await program.methods
      .initializeEscrow(cancelAmount)
      .accounts({
        initializer: initializer.publicKey,
        mint: mint,
        initializerTokenAccount: initializerTokenAccount,
        escrowState: escrowState,
        vaultAuthority: vaultAuthority,
        vaultTokenAccount: vaultTokenAccount,
        systemProgram: SystemProgram.programId,
        tokenProgram: TOKEN_PROGRAM_ID,
        rent: SYSVAR_RENT_PUBKEY,
      })
      .signers([initializer])
      .rpc();

    // Get initializer balance before cancel
    const initializerAccountBefore = await getAccount(
      provider.connection,
      initializerTokenAccount
    );

    // Cancel escrow
    await program.methods
      .cancelEscrow()
      .accounts({
        initializer: initializer.publicKey,
        escrowState: escrowState,
        vaultAuthority: vaultAuthority,
        vaultTokenAccount: vaultTokenAccount,
        initializerTokenAccount: initializerTokenAccount,
        tokenProgram: TOKEN_PROGRAM_ID,
      })
      .signers([initializer])
      .rpc();

    // Verify tokens returned to initializer
    const initializerAccountAfter = await getAccount(
      provider.connection,
      initializerTokenAccount
    );
    expect(initializerAccountAfter.amount).to.equal(
      initializerAccountBefore.amount + BigInt(cancelAmount.toNumber())
    );

    console.log("✅ Escrow canceled successfully!");
  });
});
```

### 🚀 Running the Tests

```bash
# Build the program
anchor build

# Run tests (starts local validator automatically)
anchor test

# Or run tests with logs
anchor test -- --features=verbose
```

### 📊 Expected Output

```text
token-escrow
🔄 Initializing escrow...
Initial balance: 2000000000
Program log: Initializing escrow for 1000000000 tokens
Program log: Escrow state created with PDA: 8x7...abc
Program log: Transferred 1000000000 tokens to escrow vault
  ✓ Initializes escrow (1234ms)

🔄 Completing escrow...
Receiver balance before: 0
Program log: Completing escrow transaction  
Program log: Transferred 1000000000 tokens to receiver
Program log: Closed vault token account, rent refunded to initializer
  ✓ Completes escrow (987ms)

🔄 Testing cancel escrow...
Program log: Initializing escrow for 500000000 tokens
Program log: Canceling escrow, refunding to initializer
Program log: Refunded 500000000 tokens to initializer
Program log: Closed vault token account, rent refunded to initializer
  ✓ Can cancel escrow (654ms)

3 passing (4s)
```

---

## 🧠 Client-Side Learning Points

**1. PDA Derivation**
```typescript
[escrowState] = PublicKey.findProgramAddressSync(
  [Buffer.from("escrow"), initializer.publicKey.toBuffer(), mint.toBuffer()],
  program.programId
);
```

**2. Account Verification**
```typescript
const escrowAccount = await program.account.escrowState.fetch(escrowState);
const vaultAccount = await getAccount(provider.connection, vaultTokenAccount);
```

**3. Transaction Building**
```typescript
await program.methods
  .initializeEscrow(escrowAmount)
  .accounts({ /* all required accounts */ })
  .signers([initializer])
  .rpc();
```

**4. Error Handling**
```typescript
try {
  await program.account.escrowState.fetch(escrowState);
  expect.fail("Account should be closed");
} catch (error) {
  expect(error.message).to.include("Account does not exist");
}
```

---

## 🎯 What's Next?

In [Module 4](../module-4/README.md), we'll build the client without Anchor's TypeScript SDK - using raw `@solana/web3.js` to understand exactly what's happening under the hood.
