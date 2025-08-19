# 🔐 Module 10: Solana Program Security (Sealevel Gotchas)

**Goal:** Understand Solana's unique security considerations and common pitfalls that arise from its parallel execution model and account-based architecture.

---

## 🎯 What You'll Learn

- ✅ Solana's parallel execution model and its security implications
- ✅ Account constraints and limitations that affect program design
- ✅ Cross-program invocation (CPI) safety considerations
- ✅ Reentrancy patterns - why they're different on Solana
- ✅ Anchor account constraints vs manual validation trade-offs
- ✅ Common attack vectors and how to prevent them
- ✅ Best practices for secure Solana program development

---

## 🏃‍♂️ Solana's Parallel Execution Model

Unlike Ethereum's sequential execution, Solana runs transactions in parallel using the **Sealevel** runtime.

### How Sealevel Works

```rust
// Solana analyzes account dependencies BEFORE execution
Transaction A: Modifies accounts [X, Y]
Transaction B: Modifies accounts [Z, W]  
Transaction C: Modifies accounts [Y, Z]

// Execution order:
// A and B can run in parallel (no shared accounts)
// C must wait for both A and B (shares Y with A, Z with B)
```

### Security Implications

**1. Account Locking**
- Solana locks accounts during transaction execution
- No two transactions can modify the same account simultaneously
- This prevents many traditional race conditions

**2. Transaction Ordering**
- Parallel execution means transaction order isn't guaranteed
- Don't rely on transaction sequence for security logic

---

## 🔒 Account Constraints and Limitations

### Account Size Limitations

```rust
// Maximum account size is 10MB
const MAX_ACCOUNT_SIZE: usize = 10 * 1024 * 1024;

#[account]
pub struct LargeAccount {
    pub data: Vec<u8>, // Can grow up to ~10MB
}

// Dangerous: Unbounded growth
pub fn add_data(ctx: Context<AddData>, new_data: Vec<u8>) -> Result<()> {
    let account = &mut ctx.accounts.large_account;
    account.data.extend(new_data); // Could exceed 10MB limit
    Ok(())
}

// Safe: Check size limits
pub fn add_data_safe(ctx: Context<AddData>, new_data: Vec<u8>) -> Result<()> {
    let account = &mut ctx.accounts.large_account;
    
    require!(
        account.data.len() + new_data.len() <= MAX_ACCOUNT_SIZE - 1000, // Leave buffer
        ErrorCode::AccountTooLarge
    );
    
    account.data.extend(new_data);
    Ok(())
}
```

### Rent Considerations

```rust
#[derive(Accounts)]
pub struct CreateAccount<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + 32 + 8, // Discriminator + pubkey + u64
        // Dangerous: Not rent exempt
    )]
    pub new_account: Account<'info, MyAccount>,
    
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

// Better: Ensure rent exemption
#[account(
    init,
    payer = user,
    space = 8 + 32 + 8,
    rent_exempt = enforce, // Ensures account is rent exempt
)]
pub new_account: Account<'info, MyAccount>,
```

---

## 🔄 Cross-Program Invocation (CPI) Safety

### CPI Privilege Escalation

```rust
// Dangerous: Unvalidated CPI
pub fn dangerous_cpi(ctx: Context<DangerousCPI>) -> Result<()> {
    // Attacker could pass malicious program_id
    let cpi_ctx = CpiContext::new(
        ctx.accounts.target_program.to_account_info(),
        TransferTokens {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        }
    );
    
    // This could call ANY program, not just token program!
    token_program::cpi::transfer(cpi_ctx, amount)?;
    Ok(())
}

// Safe: Validate program ID
#[derive(Accounts)]
pub struct SafeCPI<'info> {
    #[account(
        address = spl_token::ID @ ErrorCode::InvalidTokenProgram
    )]
    pub token_program: Program<'info, Token>,
    // ... other accounts
}
```

### CPI Signer Seeds Exposure

```rust
// Dangerous: Exposing seeds in CPI
pub fn dangerous_seeds_cpi(ctx: Context<SeedsCPI>) -> Result<()> {
    let seeds = &[
        b"authority",
        ctx.accounts.user.key().as_ref(),
        &[bump], // bump is exposed
    ];
    
    // If this CPI fails and reverts, the bump seed is still exposed
    // in transaction logs, potentially allowing replay attacks
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.external_program.to_account_info(),
        ExternalCall { /* accounts */ },
        &[seeds]
    );
    
    external_program::cpi::some_function(cpi_ctx)?;
    Ok(())
}

// Better: Use derived signers carefully
pub fn safe_seeds_cpi(ctx: Context<SeedsCPI>) -> Result<()> {
    // Validate the operation BEFORE exposing seeds
    require!(
        ctx.accounts.user.key() == expected_authority,
        ErrorCode::UnauthorizedAuthority
    );
    
    let seeds = &[
        b"authority",
        ctx.accounts.user.key().as_ref(),
        &[bump],
    ];
    
    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.external_program.to_account_info(),
        ExternalCall { /* accounts */ },
        &[seeds]
    );
    
    external_program::cpi::some_function(cpi_ctx)?;
    Ok(())
}
```

---

## 🔄 Reentrancy - Different on Solana

### Why Traditional Reentrancy is Harder

```rust
// Traditional Ethereum reentrancy (NOT possible on Solana)
pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    let user_balance = ctx.accounts.user_account.balance;
    
    // On Ethereum, this external call could re-enter
    // On Solana, accounts are locked during transaction
    external_program::cpi::transfer(/* ... */)?;
    
    // This update happens in the same transaction
    ctx.accounts.user_account.balance = user_balance - amount;
    Ok(())
}
```

### Solana's Account-Based Reentrancy

```rust
// Potential issue: Cross-account state inconsistency
#[derive(Accounts)]
pub struct CrossAccountUpdate<'info> {
    #[account(mut)]
    pub account_a: Account<'info, StateA>,
    #[account(mut)]
    pub account_b: Account<'info, StateB>,
}

pub fn update_linked_accounts(
    ctx: Context<CrossAccountUpdate>,
    value: u64,
) -> Result<()> {
    // Update account A
    ctx.accounts.account_a.value = value;
    
    // CPI that might fail
    external_program::cpi::risky_operation(/* ... */)?;
    
    // If CPI fails, account A is updated but account B is not
    // This can leave accounts in inconsistent state
    ctx.accounts.account_b.linked_value = value;
    
    Ok(())
}

// Better: Update all state atomically at the end
pub fn atomic_update_linked_accounts(
    ctx: Context<CrossAccountUpdate>,
    value: u64,
) -> Result<()> {
    // Perform all risky operations first
    external_program::cpi::risky_operation(/* ... */)?;
    
    // Update all accounts atomically at the end
    ctx.accounts.account_a.value = value;
    ctx.accounts.account_b.linked_value = value;
    
    Ok(())
}
```

---

## ⚖️ Anchor Constraints vs Manual Validation

### Over-Reliance on Anchor Constraints

```rust
// Potentially dangerous: Complex constraint logic
#[derive(Accounts)]
#[instruction(amount: u64)]
pub struct ComplexTransfer<'info> {
    #[account(
        mut,
        has_one = owner,
        constraint = account.balance >= amount @ ErrorCode::InsufficientFunds,
        constraint = account.active @ ErrorCode::AccountInactive,
        constraint = Clock::get()?.unix_timestamp < account.expires_at @ ErrorCode::AccountExpired,
    )]
    pub account: Account<'info, UserAccount>,
    
    pub owner: Signer<'info>,
}

// Better: Mix constraints with explicit validation
#[derive(Accounts)]
pub struct ExplicitTransfer<'info> {
    #[account(mut, has_one = owner)]
    pub account: Account<'info, UserAccount>,
    
    pub owner: Signer<'info>,
}

pub fn explicit_transfer(
    ctx: Context<ExplicitTransfer>,
    amount: u64,
) -> Result<()> {
    let account = &ctx.accounts.account;
    
    // Explicit validations are more readable and debuggable
    require!(account.active, ErrorCode::AccountInactive);
    require!(account.balance >= amount, ErrorCode::InsufficientFunds);
    require!(
        Clock::get()?.unix_timestamp < account.expires_at,
        ErrorCode::AccountExpired
    );
    
    // Perform transfer logic
    perform_transfer(ctx, amount)?;
    Ok(())
}
```

### Account Initialization Race Conditions

```rust
// Dangerous: Check-then-act pattern
pub fn initialize_if_needed(ctx: Context<InitAccount>) -> Result<()> {
    if ctx.accounts.account.data_is_empty() {
        // Race condition: Another transaction might initialize 
        // this account between the check and initialization
        initialize_account(ctx)?;
    }
    Ok(())
}

// Better: Use Anchor's init constraint
#[derive(Accounts)]
pub struct InitAccount<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + 64,
        seeds = [b"account", user.key().as_ref()],
        bump
    )]
    pub account: Account<'info, MyAccount>,
    
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

---

## 🚨 Common Attack Vectors

### 1. Fake Program ID Attacks

```rust
// Vulnerable: Accepting arbitrary program IDs
#[derive(Accounts)]
pub struct VulnerableAccounts<'info> {
    /// CHECK: This account is not validated
    pub external_program: AccountInfo<'info>,
}

pub fn call_external(ctx: Context<VulnerableAccounts>) -> Result<()> {
    // Attacker can pass malicious program
    let cpi_ctx = CpiContext::new(
        ctx.accounts.external_program.to_account_info(),
        /* accounts */
    );
    
    // This might call attacker's program instead of expected one
    Ok(())
}

// Safe: Validate program IDs
#[derive(Accounts)]
pub struct SafeAccounts<'info> {
    #[account(
        address = EXPECTED_PROGRAM_ID @ ErrorCode::InvalidProgram
    )]
    pub external_program: Program<'info, ExternalProgram>,
}
```

### 2. Account Substitution Attacks

```rust
// Vulnerable: Using unchecked account relationships
pub fn transfer_between_users(
    ctx: Context<TransferBetweenUsers>,
    amount: u64,
) -> Result<()> {
    // Attacker could substitute their own accounts
    let from = &mut ctx.accounts.from_account;
    let to = &mut ctx.accounts.to_account;
    
    from.balance -= amount;
    to.balance += amount;
    
    Ok(())
}

// Safe: Validate account ownership and relationships
#[derive(Accounts)]
pub struct SafeTransfer<'info> {
    #[account(
        mut,
        has_one = owner @ ErrorCode::InvalidOwner,
        constraint = from_account.balance >= amount @ ErrorCode::InsufficientFunds
    )]
    pub from_account: Account<'info, UserAccount>,
    
    #[account(mut)]
    pub to_account: Account<'info, UserAccount>,
    
    pub owner: Signer<'info>,
}
```

### 3. Integer Overflow/Underflow

```rust
// Dangerous: Unchecked arithmetic
pub fn unsafe_math(ctx: Context<UnsafeMath>, amount: u64) -> Result<()> {
    let account = &mut ctx.accounts.account;
    
    // Could overflow
    account.balance = account.balance + amount;
    
    // Could underflow
    account.balance = account.balance - amount;
    
    Ok(())
}

// Safe: Use checked arithmetic
pub fn safe_math(ctx: Context<SafeMath>, amount: u64) -> Result<()> {
    let account = &mut ctx.accounts.account;
    
    account.balance = account.balance
        .checked_add(amount)
        .ok_or(ErrorCode::MathOverflow)?;
    
    account.balance = account.balance
        .checked_sub(amount)
        .ok_or(ErrorCode::MathUnderflow)?;
    
    Ok(())
}
```

---

## 🛡️ Security Best Practices

### 1. Input Validation

```rust
pub fn secure_function(
    ctx: Context<SecureFunction>,
    amount: u64,
    recipient: Pubkey,
) -> Result<()> {
    // Validate inputs
    require!(amount > 0, ErrorCode::InvalidAmount);
    require!(amount <= MAX_TRANSFER_AMOUNT, ErrorCode::AmountTooLarge);
    require!(recipient != Pubkey::default(), ErrorCode::InvalidRecipient);
    
    // Additional business logic validation
    require!(
        ctx.accounts.user_account.can_transfer(amount),
        ErrorCode::TransferNotAllowed
    );
    
    // Proceed with operation
    perform_transfer(ctx, amount, recipient)?;
    Ok(())
}
```

### 2. Fail-Safe Defaults

```rust
#[account]
pub struct SecureConfig {
    pub enabled: bool,        // Default: false (disabled)
    pub max_amount: u64,      // Default: 0 (no transfers)
    pub emergency_stop: bool, // Default: false (stopped)
}

pub fn transfer_with_circuit_breaker(
    ctx: Context<TransferWithBreaker>,
    amount: u64,
) -> Result<()> {
    let config = &ctx.accounts.config;
    
    // Fail-safe: Stop if emergency activated
    require!(!config.emergency_stop, ErrorCode::EmergencyStop);
    
    // Fail-safe: Only allow if explicitly enabled
    require!(config.enabled, ErrorCode::TransfersDisabled);
    
    // Fail-safe: Respect maximum limits
    require!(amount <= config.max_amount, ErrorCode::AmountTooHigh);
    
    perform_transfer(ctx, amount)?;
    Ok(())
}
```

### 3. Comprehensive Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_security_edge_cases() {
        // Test with maximum values
        test_with_max_u64().await;
        
        // Test with zero values
        test_with_zero_amounts().await;
        
        // Test account substitution
        test_account_substitution_attack().await;
        
        // Test reentrancy scenarios
        test_cross_account_consistency().await;
        
        // Test with malicious program IDs
        test_fake_program_attack().await;
    }
    
    async fn test_account_substitution_attack() {
        let attacker_account = create_fake_account();
        
        // This should fail with InvalidOwner
        let result = call_transfer_with_fake_account(attacker_account).await;
        assert!(result.is_err());
        
        match result.unwrap_err() {
            ErrorCode::InvalidOwner => {}, // Expected
            _ => panic!("Wrong error type"),
        }
    }
}
```

---

## 🚨 Security Checklist

**Before Deploying:**

- [ ] All arithmetic operations use checked math
- [ ] All account relationships are validated
- [ ] All program IDs in CPIs are verified
- [ ] Input validation covers edge cases (0, max values)
- [ ] Account size limits are enforced
- [ ] Emergency stop mechanisms are in place
- [ ] Comprehensive test coverage including attack scenarios
- [ ] Code has been audited by security professionals
- [ ] All Anchor constraints are necessary and sufficient
- [ ] Manual validations complement (don't duplicate) constraints

**During Development:**

- [ ] Use `require!` macro for clear error messages
- [ ] Prefer explicit validation over complex constraints
- [ ] Test with realistic account sizes and data
- [ ] Monitor compute unit usage
- [ ] Use fail-safe defaults in configuration
- [ ] Document security assumptions and invariants

---

## 🎯 Key Takeaways

1. **Solana's parallel execution prevents traditional reentrancy but creates new challenges**
2. **Account locking provides safety but requires careful transaction design**
3. **CPI security is critical - validate all program IDs and account relationships**
4. **Anchor constraints are powerful but shouldn't replace explicit validation**
5. **Security testing must include edge cases and attack scenarios**
6. **Fail-safe defaults and circuit breakers are essential for production systems**

Understanding these Sealevel-specific security considerations is crucial for building robust Solana programs. The parallel execution model provides many safety guarantees, but developers must still be vigilant about the unique attack vectors and edge cases that can arise.

---

## 🎓 Conclusion

You now understand the unique security landscape of Solana development. With this knowledge, you can build more secure programs and avoid the common pitfalls that arise from Solana's innovative architecture.

Remember: **Security is not a feature to be added later - it must be designed in from the beginning.**