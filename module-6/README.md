# 🔍 Module 6: How Anchor Really Works (Not Magic)

**Goal:** Demystify Anchor by understanding what's happening under the hood. No more black boxes - you'll see exactly how discriminators, macros, and serialization work.

---

## 🎯 What You'll Learn

- ✅ **Account discriminators:** 8-byte prefixes that identify accounts
- ✅ **Instruction discriminators:** `global:<ix_name>` hashing in snake_case
- ✅ Anchor macros like `#[derive(Accounts)]`, `#[account(mut)]`, `#[instruction(args)]`
- ✅ Where your program data really lives (`account.data.borrow_mut()`)
- ✅ How Anchor generates TypeScript types from IDL
- ✅ Manual discriminator calculation and account layout

---

## 🧰 Anchor Magic Revealed

### 1. 🏷️ Account Discriminators

**What Anchor does:**
```rust
#[account]
pub struct EscrowState {
    pub initializer: Pubkey,
    pub mint: Pubkey,
    pub amount: u64,
}
```

**What actually happens:**
```rust
// Anchor generates this 8-byte discriminator
const ESCROW_STATE_DISCRIMINATOR: [u8; 8] = [215, 85, 174, 27, 98, 176, 175, 94];

// Account data layout:
// [discriminator: 8 bytes][initializer: 32 bytes][mint: 32 bytes][amount: 8 bytes]
//  Total: 80 bytes
```

**How discriminator is calculated:**
```rust
use sha2::{Digest, Sha256};

fn calculate_discriminator(name: &str) -> [u8; 8] {
    let mut hasher = Sha256::new();
    hasher.update(format!("account:{}", name).as_bytes());
    let result = hasher.finalize();
    
    let mut discriminator = [0u8; 8];
    discriminator.copy_from_slice(&result[..8]);
    discriminator
}

// calculate_discriminator("EscrowState") -> [215, 85, 174, 27, 98, 176, 175, 94]
```

### 2. 📨 Instruction Discriminators

**What Anchor does:**
```rust
pub fn initialize_escrow(ctx: Context<InitializeEscrow>, amount: u64) -> Result<()> {
    // Your logic here
}
```

**What actually happens:**
```rust
// Anchor generates this discriminator for the instruction
const INITIALIZE_ESCROW_DISCRIMINATOR: [u8; 8] = [175, 175, 109, 31, 13, 152, 155, 237];

// Calculated from: sha256("global:initialize_escrow")[..8]
fn instruction_discriminator(name: &str) -> [u8; 8] {
    let mut hasher = Sha256::new();
    hasher.update(format!("global:{}", name).as_bytes());
    let result = hasher.finalize();
    
    let mut discriminator = [0u8; 8];
    discriminator.copy_from_slice(&result[..8]);
    discriminator
}
```

### 3. 🔄 Account Validation Macros

**What you write:**
```rust
#[derive(Accounts)]
pub struct InitializeEscrow<'info> {
    #[account(
        init,
        payer = initializer,
        space = EscrowState::LEN,
        seeds = [b"escrow", initializer.key().as_ref(), mint.key().as_ref()],
        bump,
    )]
    pub escrow_state: Account<'info, EscrowState>,
    
    #[account(mut)]
    pub initializer: Signer<'info>,
}
```

**What Anchor generates:**
```rust
impl<'info> InitializeEscrow<'info> {
    pub fn __anchor_validate(&self, program_id: &Pubkey) -> Result<()> {
        // Validate initializer is a signer
        if !self.initializer.is_signer {
            return Err(anchor_lang::error::ErrorCode::AccountNotSigner.into());
        }

        // Validate PDA derivation
        let (expected_pda, bump) = Pubkey::find_program_address(
            &[b"escrow", self.initializer.key().as_ref(), self.mint.key().as_ref()],
            program_id,
        );
        if self.escrow_state.key() != &expected_pda {
            return Err(anchor_lang::error::ErrorCode::ConstraintSeeds.into());
        }

        // Validate account space
        if self.escrow_state.data_len() < EscrowState::LEN {
            return Err(anchor_lang::error::ErrorCode::AccountDidNotSerialize.into());
        }

        Ok(())
    }
}
```

---

## 🛠️ Manual Anchor Implementation

Let's build parts of Anchor manually to understand how it works:

### 📦 Manual Account Wrapper (`src/manual-anchor.rs`)

```rust
use anchor_lang::prelude::*;
use std::ops::{Deref, DerefMut};

// Manual implementation of what #[account] does
pub struct ManualAccount<T> {
    pub key: Pubkey,
    pub lamports: u64,
    pub data: T,
    pub owner: Pubkey,
    pub executable: bool,
    pub rent_epoch: u64,
}

impl<T: AccountSerialize + AccountDeserialize> ManualAccount<T> {
    // Manual deserialization (what Anchor does automatically)
    pub fn try_from_account_info(account_info: &AccountInfo) -> Result<Self> {
        // Check discriminator first
        let discriminator = T::discriminator();
        if account_info.data.borrow()[..8] != discriminator {
            return Err(anchor_lang::error::ErrorCode::AccountDiscriminatorMismatch.into());
        }

        // Deserialize account data (skip discriminator)
        let data_slice = &account_info.data.borrow()[8..];
        let data = T::try_deserialize(&mut &data_slice[..])?;

        Ok(ManualAccount {
            key: *account_info.key,
            lamports: account_info.lamports(),
            data,
            owner: *account_info.owner,
            executable: account_info.executable,
            rent_epoch: account_info.rent_epoch,
        })
    }

    // Manual serialization (what Anchor does on account close/update)
    pub fn try_serialize_to_account_info(&self, account_info: &AccountInfo) -> Result<()> {
        let mut data = account_info.data.borrow_mut();
        
        // Write discriminator
        data[..8].copy_from_slice(&T::discriminator());
        
        // Write account data
        let mut writer = &mut data[8..];
        self.data.try_serialize(&mut writer)?;
        
        Ok(())
    }
}

impl<T> Deref for ManualAccount<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.data
    }
}

impl<T> DerefMut for ManualAccount<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.data
    }
}

// Manual implementation of AccountSerialize/Deserialize
pub trait ManualAccountSerialize {
    fn try_serialize<W: std::io::Write>(&self, writer: &mut W) -> Result<()>;
    fn discriminator() -> [u8; 8];
}

pub trait ManualAccountDeserialize: Sized {
    fn try_deserialize<R: std::io::Read>(reader: &mut R) -> Result<Self>;
}

// Example implementation for EscrowState
impl ManualAccountSerialize for EscrowState {
    fn try_serialize<W: std::io::Write>(&self, writer: &mut W) -> Result<()> {
        // Manual borsh serialization
        writer.write_all(&self.initializer.to_bytes())?;
        writer.write_all(&self.mint.to_bytes())?;
        writer.write_all(&self.amount.to_le_bytes())?;
        writer.write_all(&[self.vault_authority_bump])?;
        Ok(())
    }

    fn discriminator() -> [u8; 8] {
        // Calculated: sha256("account:EscrowState")[..8]
        [215, 85, 174, 27, 98, 176, 175, 94]
    }
}

impl ManualAccountDeserialize for EscrowState {
    fn try_deserialize<R: std::io::Read>(reader: &mut R) -> Result<Self> {
        let mut initializer_bytes = [0u8; 32];
        let mut mint_bytes = [0u8; 32];
        let mut amount_bytes = [0u8; 8];
        let mut bump_bytes = [0u8; 1];

        reader.read_exact(&mut initializer_bytes)?;
        reader.read_exact(&mut mint_bytes)?;
        reader.read_exact(&mut amount_bytes)?;
        reader.read_exact(&mut bump_bytes)?;

        Ok(EscrowState {
            initializer: Pubkey::new_from_array(initializer_bytes),
            mint: Pubkey::new_from_array(mint_bytes),
            amount: u64::from_le_bytes(amount_bytes),
            vault_authority_bump: bump_bytes[0],
        })
    }
}
```

### 🎛️ Manual Context Implementation

```rust
// Manual implementation of what Context<T> does
pub struct ManualContext<'info, T> {
    pub program_id: &'info Pubkey,
    pub accounts: T,
    pub remaining_accounts: &'info [AccountInfo<'info>],
    pub bumps: std::collections::HashMap<String, u8>,
}

impl<'info, T> ManualContext<'info, T>
where
    T: Accounts<'info>,
{
    pub fn new(
        program_id: &'info Pubkey,
        accounts: &'info [AccountInfo<'info>],
        instruction_data: &[u8],
    ) -> Result<Self> {
        // Parse accounts (what #[derive(Accounts)] generates)
        let accounts = T::try_accounts(program_id, accounts, instruction_data)?;

        // Collect PDA bumps
        let bumps = T::collect_bumps(program_id, accounts);

        Ok(ManualContext {
            program_id,
            accounts,
            remaining_accounts: &[], // Simplified
            bumps,
        })
    }
}

// Manual implementation of what #[derive(Accounts)] generates
pub trait ManualAccounts<'info>: Sized {
    fn try_accounts(
        program_id: &Pubkey,
        accounts: &'info [AccountInfo<'info>],
        instruction_data: &[u8],
    ) -> Result<Self>;

    fn collect_bumps(program_id: &Pubkey, accounts: &Self) -> std::collections::HashMap<String, u8>;
}

// Example manual implementation for InitializeEscrow
impl<'info> ManualAccounts<'info> for InitializeEscrow<'info> {
    fn try_accounts(
        program_id: &Pubkey,
        accounts: &'info [AccountInfo<'info>],
        _instruction_data: &[u8],
    ) -> Result<Self> {
        let mut account_iter = accounts.iter();

        // Parse each account in order
        let initializer = next_account_info(&mut account_iter)?;
        if !initializer.is_signer {
            return Err(ErrorCode::AccountNotSigner.into());
        }

        let mint = next_account_info(&mut account_iter)?;
        let initializer_token_account = next_account_info(&mut account_iter)?;
        let escrow_state = next_account_info(&mut account_iter)?;
        
        // Validate PDA
        let (expected_pda, bump) = Pubkey::find_program_address(
            &[b"escrow", initializer.key.as_ref(), mint.key.as_ref()],
            program_id,
        );
        if escrow_state.key != &expected_pda {
            return Err(ErrorCode::ConstraintSeeds.into());
        }

        // Continue parsing remaining accounts...
        let vault_authority = next_account_info(&mut account_iter)?;
        let vault_token_account = next_account_info(&mut account_iter)?;
        let system_program = next_account_info(&mut account_iter)?;
        let token_program = next_account_info(&mut account_iter)?;
        let rent = next_account_info(&mut account_iter)?;

        Ok(InitializeEscrow {
            initializer: Signer::try_from(initializer)?,
            mint: Account::try_from(mint)?,
            initializer_token_account: Account::try_from(initializer_token_account)?,
            escrow_state: Account::try_from(escrow_state)?,
            vault_authority: UncheckedAccount::try_from(vault_authority)?,
            vault_token_account: Account::try_from(vault_token_account)?,
            system_program: Program::try_from(system_program)?,
            token_program: Program::try_from(token_program)?,
            rent: Sysvar::try_from(rent)?,
        })
    }

    fn collect_bumps(_program_id: &Pubkey, accounts: &Self) -> std::collections::HashMap<String, u8> {
        let mut bumps = std::collections::HashMap::new();
        // Collect all PDA bumps used in account validation
        bumps.insert("escrow_state".to_string(), accounts.escrow_state.bump);
        bumps.insert("vault_authority".to_string(), accounts.vault_authority.bump);
        bumps.insert("vault_token_account".to_string(), accounts.vault_token_account.bump);
        bumps
    }
}
```

---

## 🔬 Understanding IDL Generation

**What Anchor generates for TypeScript:**
```json
{
  "version": "0.1.0",
  "name": "token_escrow",
  "instructions": [
    {
      "name": "initializeEscrow",
      "accounts": [
        {"name": "initializer", "isMut": true, "isSigner": true},
        {"name": "mint", "isMut": false, "isSigner": false},
        {"name": "initializerTokenAccount", "isMut": true, "isSigner": false}
      ],
      "args": [
        {"name": "amount", "type": "u64"}
      ]
    }
  ],
  "accounts": [
    {
      "name": "EscrowState",
      "type": {
        "kind": "struct",
        "fields": [
          {"name": "initializer", "type": "publicKey"},
          {"name": "mint", "type": "publicKey"},
          {"name": "amount", "type": "u64"},
          {"name": "vaultAuthorityBump", "type": "u8"}
        ]
      }
    }
  ],
  "errors": [
    {"code": 6000, "name": "InvalidMint", "msg": "Invalid mint for token account"}
  ]
}
```

**How TypeScript types are generated:**
```typescript
// Generated from IDL
export type TokenEscrow = {
  "version": "0.1.0",
  "name": "token_escrow",
  "instructions": [
    {
      "name": "initializeEscrow",
      "accounts": [
        {"name": "initializer", "isMut": true, "isSigner": true, "docs": []},
        // ...
      ],
      "args": [
        {"name": "amount", "type": "u64"}
      ]
    }
  ],
  "accounts": [
    {
      "name": "escrowState",
      "type": {
        "kind": "struct",
        "fields": [
          {"name": "initializer", "type": "publicKey"},
          {"name": "mint", "type": "publicKey"},
          {"name": "amount", "type": "u64"},
          {"name": "vaultAuthorityBump", "type": "u8"}
        ]
      }
    }
  ]
};

// TypeScript account type
export type EscrowState = {
  initializer: PublicKey;
  mint: PublicKey;
  amount: anchor.BN;
  vaultAuthorityBump: number;
};
```

---

## 🧪 Testing Discriminator Generation

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sha2::{Digest, Sha256};

    #[test]
    fn test_account_discriminator() {
        fn discriminator(name: &str) -> [u8; 8] {
            let mut hasher = Sha256::new();
            hasher.update(format!("account:{}", name).as_bytes());
            let result = hasher.finalize();
            let mut disc = [0u8; 8];
            disc.copy_from_slice(&result[..8]);
            disc
        }

        // Test that our manual calculation matches Anchor's
        let manual_disc = discriminator("EscrowState");
        let anchor_disc = EscrowState::discriminator();
        
        assert_eq!(manual_disc, anchor_disc);
        println!("✅ EscrowState discriminator: {:?}", manual_disc);
    }

    #[test]
    fn test_instruction_discriminator() {
        fn instruction_discriminator(name: &str) -> [u8; 8] {
            let mut hasher = Sha256::new();
            hasher.update(format!("global:{}", name).as_bytes());
            let result = hasher.finalize();
            let mut disc = [0u8; 8];
            disc.copy_from_slice(&result[..8]);
            disc
        }

        // Test instruction discriminators
        let init_disc = instruction_discriminator("initialize_escrow");
        println!("✅ initialize_escrow discriminator: {:?}", init_disc);
        
        let complete_disc = instruction_discriminator("complete_escrow");
        println!("✅ complete_escrow discriminator: {:?}", complete_disc);
    }

    #[test] 
    fn test_account_layout() {
        // Test that account size calculation is correct
        let expected_size = 8 + 32 + 32 + 8 + 1; // discriminator + initializer + mint + amount + bump
        assert_eq!(EscrowState::LEN, expected_size);
        
        println!("✅ EscrowState account layout:");
        println!("  - Discriminator: 8 bytes");
        println!("  - Initializer:   32 bytes");
        println!("  - Mint:          32 bytes");
        println!("  - Amount:        8 bytes");
        println!("  - Bump:          1 byte");
        println!("  - Total:         {} bytes", expected_size);
    }
}
```

---

## 🧠 Key Takeaways

**1. No Magic - Just Hashing**
```rust
// Account discriminator = sha256("account:AccountName")[..8]
// Instruction discriminator = sha256("global:instruction_name")[..8]
```

**2. Account Data Layout**
```
[8-byte discriminator][serialized account data]
```

**3. Anchor Macros Generate Code**
```rust
#[derive(Accounts)] // -> Generates validation logic
#[account(mut)]     // -> Generates mutability checks  
#[instruction(args)] // -> Generates instruction parsing
```

**4. TypeScript Types From IDL**
```typescript
// IDL describes your program's interface
// anchor build generates target/idl/program.json
// TypeScript types generated from IDL
```

**5. Manual Account Handling**
```rust
// You can implement everything Anchor does manually
// Useful for understanding and custom behavior
let data = &account_info.data.borrow()[8..]; // Skip discriminator
let state = EscrowState::deserialize(data)?;
```

---

## 🎯 What's Next?

In [Module 7](../module-7/README.md), we'll strip away Anchor completely and rewrite our escrow program using raw Solana - no macros, manual validation, and direct `invoke_signed` calls.