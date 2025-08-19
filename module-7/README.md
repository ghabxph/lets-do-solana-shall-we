# 🛠️ Module 7: Rewriting It Without Anchor

**Goal:** Strip away Anchor and implement the same escrow program using raw Solana. No macros, manual deserialization, hand-crafted PDA validation, and direct `invoke_signed` calls.

---

## 🎯 What You'll Learn

- ✅ Raw Solana program structure without Anchor
- ✅ Manual instruction parsing and data deserialization
- ✅ Hand-crafted PDA validation and bump generation
- ✅ Direct `invoke_signed` usage for cross-program invocation
- ✅ Manual Borsh serialization/deserialization
- ✅ Raw account validation and constraint checking
- ✅ Understanding the true cost and benefit of frameworks

---

## 🏗️ Project Structure

```bash
# Create raw Solana program
mkdir raw-escrow-program
cd raw-escrow-program

# Create Cargo.toml
cat > Cargo.toml << EOF
[package]
name = "raw-escrow-program"
version = "0.1.0"
edition = "2021"

[dependencies]
solana-program = "~1.16"
borsh = "0.10"
thiserror = "1.0"

[lib]
crate-type = ["cdylib", "lib"]
EOF

mkdir src
```

---

## 🦀 Raw Solana Implementation

### 📁 Main Program (`src/lib.rs`)

```rust
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program::{invoke, invoke_signed},
    program_error::ProgramError,
    program_pack::{IsInitialized, Pack, Sealed},
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::{rent, Sysvar},
};
use borsh::{BorshDeserialize, BorshSerialize};
use std::convert::TryInto;

// Program entrypoint
entrypoint!(process_instruction);

// Program ID (would be replaced with actual deployed program ID)
solana_program::declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

// Manual instruction enum (what Anchor generates automatically)
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone)]
pub enum EscrowInstruction {
    /// Initialize escrow account
    /// Accounts:
    /// 0. [signer] Initializer
    /// 1. [writable] Initializer's token account
    /// 2. [] Mint account
    /// 3. [writable] Escrow state account (PDA)
    /// 4. [] Vault authority (PDA)
    /// 5. [writable] Vault token account (PDA)
    /// 6. [] System program
    /// 7. [] Token program
    /// 8. [] Rent sysvar
    InitializeEscrow {
        amount: u64,
    },

    /// Complete escrow transaction
    /// Accounts:
    /// 0. [signer] Receiver
    /// 1. [writable] Initializer
    /// 2. [writable] Escrow state account
    /// 3. [] Vault authority (PDA)
    /// 4. [writable] Vault token account
    /// 5. [writable] Receiver's token account
    /// 6. [] Token program
    CompleteEscrow,

    /// Cancel escrow and return tokens to initializer
    /// Accounts:
    /// 0. [signer] Initializer
    /// 1. [writable] Escrow state account
    /// 2. [] Vault authority (PDA)
    /// 3. [writable] Vault token account
    /// 4. [writable] Initializer's token account
    /// 5. [] Token program
    CancelEscrow,
}

// Manual state struct (what #[account] generates)
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone)]
pub struct EscrowState {
    pub initializer: Pubkey,
    pub mint: Pubkey,
    pub amount: u64,
    pub vault_authority_bump: u8,
}

impl Sealed for EscrowState {}

impl IsInitialized for EscrowState {
    fn is_initialized(&self) -> bool {
        self.initializer != Pubkey::default()
    }
}

impl Pack for EscrowState {
    const LEN: usize = 32 + 32 + 8 + 1; // 73 bytes

    fn pack_into_slice(&self, dst: &mut [u8]) {
        let data = self.try_to_vec().unwrap();
        dst[..data.len()].copy_from_slice(&data);
    }

    fn unpack_from_slice(src: &[u8]) -> Result<Self, ProgramError> {
        Self::try_from_slice(src).map_err(|_| ProgramError::InvalidAccountData)
    }
}

// Custom error types (what #[error_code] generates)
#[derive(thiserror::Error, Debug, Clone)]
pub enum EscrowError {
    #[error("Invalid instruction")]
    InvalidInstruction,
    #[error("Account not signer")]
    AccountNotSigner,
    #[error("Invalid mint")]
    InvalidMint,
    #[error("Invalid owner")]
    InvalidOwner,
    #[error("Insufficient tokens")]
    InsufficientTokens,
    #[error("Invalid PDA")]
    InvalidPDA,
    #[error("Account already initialized")]
    AccountAlreadyInitialized,
}

impl From<EscrowError> for ProgramError {
    fn from(e: EscrowError) -> Self {
        ProgramError::Custom(e as u32)
    }
}

// Main program processor
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    msg!("Raw escrow program entrypoint");

    // Manual instruction deserialization (what Anchor does automatically)
    let instruction = EscrowInstruction::try_from_slice(instruction_data)
        .map_err(|_| EscrowError::InvalidInstruction)?;

    match instruction {
        EscrowInstruction::InitializeEscrow { amount } => {
            process_initialize_escrow(program_id, accounts, amount)
        }
        EscrowInstruction::CompleteEscrow => {
            process_complete_escrow(program_id, accounts)
        }
        EscrowInstruction::CancelEscrow => {
            process_cancel_escrow(program_id, accounts)
        }
    }
}

// Initialize escrow implementation
fn process_initialize_escrow(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    amount: u64,
) -> ProgramResult {
    msg!("Processing InitializeEscrow with amount: {}", amount);

    // Manual account parsing (what #[derive(Accounts)] does)
    let accounts_iter = &mut accounts.iter();
    
    let initializer = next_account_info(accounts_iter)?;
    let initializer_token_account = next_account_info(accounts_iter)?;
    let mint = next_account_info(accounts_iter)?;
    let escrow_state = next_account_info(accounts_iter)?;
    let vault_authority = next_account_info(accounts_iter)?;
    let vault_token_account = next_account_info(accounts_iter)?;
    let system_program = next_account_info(accounts_iter)?;
    let token_program = next_account_info(accounts_iter)?;
    let rent_sysvar = next_account_info(accounts_iter)?;

    // Manual validation (what Anchor macros do)
    if !initializer.is_signer {
        return Err(EscrowError::AccountNotSigner.into());
    }

    // Validate PDA derivation manually
    let escrow_seeds = &[
        b"escrow",
        initializer.key.as_ref(),
        mint.key.as_ref(),
    ];
    let (expected_escrow_state, escrow_bump) = 
        Pubkey::find_program_address(escrow_seeds, program_id);
    
    if escrow_state.key != &expected_escrow_state {
        msg!("Invalid escrow state PDA. Expected: {}, Got: {}", 
             expected_escrow_state, escrow_state.key);
        return Err(EscrowError::InvalidPDA.into());
    }

    // Validate vault authority PDA
    let vault_authority_seeds = &[
        b"vault-authority",
        escrow_state.key.as_ref(),
    ];
    let (expected_vault_authority, vault_authority_bump) = 
        Pubkey::find_program_address(vault_authority_seeds, program_id);
    
    if vault_authority.key != &expected_vault_authority {
        return Err(EscrowError::InvalidPDA.into());
    }

    // Validate vault token account PDA
    let vault_seeds = &[
        b"vault",
        escrow_state.key.as_ref(),
    ];
    let (expected_vault_token_account, _vault_bump) = 
        Pubkey::find_program_address(vault_seeds, program_id);
    
    if vault_token_account.key != &expected_vault_token_account {
        return Err(EscrowError::InvalidPDA.into());
    }

    let rent = Rent::from_account_info(rent_sysvar)?;

    // Create escrow state account manually
    let escrow_state_size = EscrowState::LEN;
    let escrow_rent_lamports = rent.minimum_balance(escrow_state_size);

    invoke_signed(
        &system_instruction::create_account(
            initializer.key,
            escrow_state.key,
            escrow_rent_lamports,
            escrow_state_size as u64,
            program_id,
        ),
        &[
            initializer.clone(),
            escrow_state.clone(),
            system_program.clone(),
        ],
        &[&[
            b"escrow",
            initializer.key.as_ref(),
            mint.key.as_ref(),
            &[escrow_bump],
        ]],
    )?;

    // Initialize escrow state data manually
    let escrow_data = EscrowState {
        initializer: *initializer.key,
        mint: *mint.key,
        amount,
        vault_authority_bump,
    };

    EscrowState::pack(escrow_data, &mut escrow_state.try_borrow_mut_data()?)?;
    msg!("Escrow state initialized");

    // Create vault token account manually (using CPI)
    let create_vault_ix = spl_token::instruction::initialize_account(
        token_program.key,
        vault_token_account.key,
        mint.key,
        vault_authority.key,
    )?;

    // First create the account
    invoke_signed(
        &system_instruction::create_account(
            initializer.key,
            vault_token_account.key,
            rent.minimum_balance(spl_token::state::Account::LEN),
            spl_token::state::Account::LEN as u64,
            token_program.key,
        ),
        &[
            initializer.clone(),
            vault_token_account.clone(),
            system_program.clone(),
        ],
        &[&[
            b"vault",
            escrow_state.key.as_ref(),
            &[_vault_bump],
        ]],
    )?;

    // Then initialize it as a token account
    invoke(
        &create_vault_ix,
        &[
            vault_token_account.clone(),
            mint.clone(),
            vault_authority.clone(),
            rent_sysvar.clone(),
            token_program.clone(),
        ],
    )?;

    // Transfer tokens from initializer to vault (manual CPI)
    let transfer_ix = spl_token::instruction::transfer(
        token_program.key,
        initializer_token_account.key,
        vault_token_account.key,
        initializer.key,
        &[],
        amount,
    )?;

    invoke(
        &transfer_ix,
        &[
            initializer_token_account.clone(),
            vault_token_account.clone(),
            initializer.clone(),
            token_program.clone(),
        ],
    )?;

    msg!("Transferred {} tokens to vault", amount);
    Ok(())
}

// Complete escrow implementation
fn process_complete_escrow(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    msg!("Processing CompleteEscrow");

    let accounts_iter = &mut accounts.iter();
    
    let receiver = next_account_info(accounts_iter)?;
    let initializer = next_account_info(accounts_iter)?;
    let escrow_state = next_account_info(accounts_iter)?;
    let vault_authority = next_account_info(accounts_iter)?;
    let vault_token_account = next_account_info(accounts_iter)?;
    let receiver_token_account = next_account_info(accounts_iter)?;
    let token_program = next_account_info(accounts_iter)?;

    if !receiver.is_signer {
        return Err(EscrowError::AccountNotSigner.into());
    }

    // Load and validate escrow state manually
    let escrow_data = EscrowState::unpack(&escrow_state.try_borrow_data()?)?;
    
    if escrow_data.initializer != *initializer.key {
        return Err(EscrowError::InvalidOwner.into());
    }

    // Validate vault authority PDA
    let vault_authority_seeds = &[
        b"vault-authority",
        escrow_state.key.as_ref(),
    ];
    let (expected_vault_authority, _bump) = 
        Pubkey::find_program_address(vault_authority_seeds, program_id);
    
    if vault_authority.key != &expected_vault_authority {
        return Err(EscrowError::InvalidPDA.into());
    }

    // Transfer tokens from vault to receiver using invoke_signed
    let transfer_ix = spl_token::instruction::transfer(
        token_program.key,
        vault_token_account.key,
        receiver_token_account.key,
        vault_authority.key,
        &[],
        escrow_data.amount,
    )?;

    invoke_signed(
        &transfer_ix,
        &[
            vault_token_account.clone(),
            receiver_token_account.clone(),
            vault_authority.clone(),
            token_program.clone(),
        ],
        &[&[
            b"vault-authority",
            escrow_state.key.as_ref(),
            &[escrow_data.vault_authority_bump],
        ]],
    )?;

    msg!("Transferred {} tokens to receiver", escrow_data.amount);

    // Close vault token account manually
    let close_vault_ix = spl_token::instruction::close_account(
        token_program.key,
        vault_token_account.key,
        initializer.key,
        vault_authority.key,
        &[],
    )?;

    invoke_signed(
        &close_vault_ix,
        &[
            vault_token_account.clone(),
            initializer.clone(),
            vault_authority.clone(),
            token_program.clone(),
        ],
        &[&[
            b"vault-authority",
            escrow_state.key.as_ref(),
            &[escrow_data.vault_authority_bump],
        ]],
    )?;

    // Close escrow state account manually (transfer lamports to initializer)
    let escrow_lamports = escrow_state.lamports();
    **escrow_state.try_borrow_mut_lamports()? = 0;
    **initializer.try_borrow_mut_lamports()? += escrow_lamports;

    msg!("Escrow completed successfully");
    Ok(())
}

// Cancel escrow implementation
fn process_cancel_escrow(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    msg!("Processing CancelEscrow");

    let accounts_iter = &mut accounts.iter();
    
    let initializer = next_account_info(accounts_iter)?;
    let escrow_state = next_account_info(accounts_iter)?;
    let vault_authority = next_account_info(accounts_iter)?;
    let vault_token_account = next_account_info(accounts_iter)?;
    let initializer_token_account = next_account_info(accounts_iter)?;
    let token_program = next_account_info(accounts_iter)?;

    if !initializer.is_signer {
        return Err(EscrowError::AccountNotSigner.into());
    }

    // Load escrow state and validate owner
    let escrow_data = EscrowState::unpack(&escrow_state.try_borrow_data()?)?;
    
    if escrow_data.initializer != *initializer.key {
        return Err(EscrowError::InvalidOwner.into());
    }

    // Transfer tokens back to initializer using invoke_signed
    let transfer_ix = spl_token::instruction::transfer(
        token_program.key,
        vault_token_account.key,
        initializer_token_account.key,
        vault_authority.key,
        &[],
        escrow_data.amount,
    )?;

    invoke_signed(
        &transfer_ix,
        &[
            vault_token_account.clone(),
            initializer_token_account.clone(),
            vault_authority.clone(),
            token_program.clone(),
        ],
        &[&[
            b"vault-authority",
            escrow_state.key.as_ref(),
            &[escrow_data.vault_authority_bump],
        ]],
    )?;

    msg!("Refunded {} tokens to initializer", escrow_data.amount);

    // Close vault token account
    let close_vault_ix = spl_token::instruction::close_account(
        token_program.key,
        vault_token_account.key,
        initializer.key,
        vault_authority.key,
        &[],
    )?;

    invoke_signed(
        &close_vault_ix,
        &[
            vault_token_account.clone(),
            initializer.clone(),
            vault_authority.clone(),
            token_program.clone(),
        ],
        &[&[
            b"vault-authority",
            escrow_state.key.as_ref(),
            &[escrow_data.vault_authority_bump],
        ]],
    )?;

    // Close escrow state account
    let escrow_lamports = escrow_state.lamports();
    **escrow_state.try_borrow_mut_lamports()? = 0;
    **initializer.try_borrow_mut_lamports()? += escrow_lamports;

    msg!("Escrow canceled successfully");
    Ok(())
}
```

---

## 🧪 Raw Solana Test (`tests/raw-escrow-test.rs`)

```rust
use solana_program::{
    instruction::{AccountMeta, Instruction},
    pubkey::Pubkey,
    system_program,
    sysvar,
};
use solana_program_test::*;
use solana_sdk::{
    account::Account,
    signature::{Keypair, Signer},
    transaction::Transaction,
};
use spl_token;
use raw_escrow_program::{EscrowInstruction, EscrowState, process_instruction, id};
use borsh::BorshSerialize;

#[tokio::test]
async fn test_raw_escrow_full_flow() {
    let program_id = id();
    
    // Create program test
    let (mut banks_client, payer, recent_blockhash) = ProgramTest::new(
        "raw_escrow_program",
        program_id,
        processor!(process_instruction),
    )
    .start()
    .await;

    // Create test accounts
    let initializer = Keypair::new();
    let receiver = Keypair::new();
    let mint = Keypair::new();
    let initializer_token_account = Keypair::new();
    let receiver_token_account = Keypair::new();

    println!("🧪 Testing raw Solana escrow program...");

    // Fund accounts, create mint, create token accounts, etc.
    // (Similar setup as previous tests but with manual instruction building)

    // Build initialize escrow instruction manually
    let (escrow_state, _) = Pubkey::find_program_address(
        &[
            b"escrow",
            initializer.pubkey().as_ref(),
            mint.pubkey().as_ref(),
        ],
        &program_id,
    );

    let (vault_authority, _) = Pubkey::find_program_address(
        &[
            b"vault-authority",
            escrow_state.as_ref(),
        ],
        &program_id,
    );

    let (vault_token_account, _) = Pubkey::find_program_address(
        &[
            b"vault",
            escrow_state.as_ref(),
        ],
        &program_id,
    );

    let escrow_amount = 1000_u64;

    let init_instruction = Instruction::new_with_borsh(
        program_id,
        &EscrowInstruction::InitializeEscrow { amount: escrow_amount },
        vec![
            AccountMeta::new(initializer.pubkey(), true),
            AccountMeta::new(initializer_token_account.pubkey(), false),
            AccountMeta::new_readonly(mint.pubkey(), false),
            AccountMeta::new(escrow_state, false),
            AccountMeta::new_readonly(vault_authority, false),
            AccountMeta::new(vault_token_account, false),
            AccountMeta::new_readonly(system_program::id(), false),
            AccountMeta::new_readonly(spl_token::id(), false),
            AccountMeta::new_readonly(sysvar::rent::id(), false),
        ],
    );

    let init_tx = Transaction::new_signed_with_payer(
        &[init_instruction],
        Some(&payer.pubkey()),
        &[&payer, &initializer],
        recent_blockhash,
    );

    banks_client.process_transaction(init_tx).await.unwrap();
    println!("✅ Raw escrow initialized");

    // Test complete escrow
    let complete_instruction = Instruction::new_with_borsh(
        program_id,
        &EscrowInstruction::CompleteEscrow,
        vec![
            AccountMeta::new(receiver.pubkey(), true),
            AccountMeta::new(initializer.pubkey(), false),
            AccountMeta::new(escrow_state, false),
            AccountMeta::new_readonly(vault_authority, false),
            AccountMeta::new(vault_token_account, false),
            AccountMeta::new(receiver_token_account.pubkey(), false),
            AccountMeta::new_readonly(spl_token::id(), false),
        ],
    );

    let complete_tx = Transaction::new_signed_with_payer(
        &[complete_instruction],
        Some(&payer.pubkey()),
        &[&payer, &receiver],
        recent_blockhash,
    );

    banks_client.process_transaction(complete_tx).await.unwrap();
    println!("✅ Raw escrow completed");

    println!("🎉 Raw Solana escrow test passed!");
}
```

---

## 📊 Comparison: Anchor vs Raw Solana

| **Feature** | **Anchor (Lines)** | **Raw Solana (Lines)** | **Complexity** |
|-------------|-------------------|------------------------|-----------------|
| Account validation | 3 lines (`#[account(mut)]`) | 20+ lines (manual checks) | 7x more complex |
| PDA derivation | Auto (`seeds`, `bump`) | Manual (`find_program_address`) | 3x more code |
| Serialization | Auto (`#[account]`) | Manual Borsh | 5x more code |
| Error handling | Auto (`#[error_code]`) | Manual error types | 4x more code |
| Instruction parsing | Auto (`Context<T>`) | Manual parsing | 6x more code |
| Cross-program calls | `CpiContext::new()` | Manual `invoke_signed()` | 2x more code |
| **Total LOC** | **~150 lines** | **~400+ lines** | **~3x more code** |

---

## 🧠 Key Learning Points

**1. Manual Account Validation**
```rust
// Every validation you write by hand
if !initializer.is_signer {
    return Err(EscrowError::AccountNotSigner.into());
}

let (expected_pda, bump) = Pubkey::find_program_address(seeds, program_id);
if account.key != &expected_pda {
    return Err(EscrowError::InvalidPDA.into());
}
```

**2. Manual invoke_signed**
```rust
// Direct cross-program invocation
invoke_signed(
    &transfer_ix,
    accounts,
    &[&[
        b"vault-authority",
        escrow_state.key.as_ref(),
        &[vault_authority_bump],
    ]],
)?;
```

**3. Manual Serialization**
```rust
// Pack/unpack account data manually
let escrow_data = EscrowState::unpack(&escrow_state.try_borrow_data()?)?;
EscrowState::pack(escrow_data, &mut escrow_state.try_borrow_mut_data()?)?;
```

**4. Error-Prone but Educational**
- More room for bugs (wrong account order, missing validations)
- Deeper understanding of Solana internals
- Full control over every operation
- No magic, just explicit code

**5. Predictability and Debugging Advantages**
- No macro magic - everything is explicit and visible
- Easier to debug since all code paths are clear
- More predictable behavior with no hidden abstractions
- Direct control over memory layout and operations

---

## 🔍 When to Use Raw Solana

**✅ Use Raw Solana When:**
- Need complete transparency and predictability in code execution
- Require fine-grained control over every operation
- Working with complex custom account layouts
- Building foundational infrastructure
- Security-critical applications requiring manual validation
- Learning Solana internals and understanding the underlying mechanics

**❌ Stick with Anchor When:**
- Rapid prototyping and development
- Standard DeFi/NFT applications
- Team development (maintainability)
- Learning Solana development
- Time-to-market is important

---

## 🎯 What's Next?

In [Module 8](../module-8/README.md), we'll go back to Anchor with our newfound understanding and appreciate the abstraction it provides - comparing development experience, security, and maintainability.