# 🚀 Module 10: Extra - Advanced Solana Development Patterns

**Goal:** Explore advanced Solana development patterns, best practices, and real-world implementation strategies that go beyond the basics.

---

## 🎯 What You'll Learn

- ✅ Advanced PDA patterns and optimization techniques  
- ✅ State compression and Merkle trees for scalable NFTs
- ✅ Cross-program invocation (CPI) best practices
- ✅ Account versioning and migration strategies
- ✅ Advanced testing patterns and security considerations
- ✅ Performance optimization and compute unit management
- ✅ Integration with off-chain services and oracles
- ✅ Deployment strategies and program upgrades

---

## 🏗️ Advanced PDA Patterns

### 1. Hierarchical PDAs

```rust
// Parent-child relationship with PDAs
let (parent_pda, _) = Pubkey::find_program_address(
    &[b"parent", user.key().as_ref()],
    program_id,
);

let (child_pda, _) = Pubkey::find_program_address(
    &[b"child", parent_pda.as_ref(), &index.to_le_bytes()],
    program_id,
);
```

### 2. Dynamic PDA Generation

```rust
// Using counters for unique PDAs
#[account]
pub struct GlobalState {
    pub counter: u64,
}

let (item_pda, _) = Pubkey::find_program_address(
    &[
        b"item", 
        user.key().as_ref(),
        &global_state.counter.to_le_bytes()
    ],
    program_id,
);
```

### 3. PDA Optimization Techniques

```rust
// Store bump seeds to avoid recalculation
#[account]
pub struct OptimizedAccount {
    pub bump: u8,
    pub authority: Pubkey,
    pub data: Vec<u8>,
}

// Use stored bump in subsequent operations
let signer_seeds = &[
    b"authority",
    self.authority.as_ref(),
    &[self.bump]
];
```

---

## 🌳 State Compression with Merkle Trees

### 1. Compressed NFT Implementation

```rust
use mpl_bubblegum::{
    instructions::{CreateTreeConfigBuilder, MintToCollectionV1Builder},
    types::MetadataArgs,
};

// Create a compressed NFT collection
pub fn create_compressed_nft_tree(
    ctx: Context<CreateTree>,
    max_depth: u32,
    max_buffer_size: u32,
) -> Result<()> {
    let tree_config = ctx.accounts.tree_config.to_account_info();
    let merkle_tree = ctx.accounts.merkle_tree.to_account_info();

    CreateTreeConfigBuilder::new()
        .tree_config(&tree_config)
        .merkle_tree(&merkle_tree)
        .payer(&ctx.accounts.payer)
        .tree_creator(&ctx.accounts.tree_creator)
        .log_wrapper(&ctx.accounts.log_wrapper)
        .compression_program(&ctx.accounts.compression_program)
        .system_program(&ctx.accounts.system_program)
        .max_depth(max_depth)
        .max_buffer_size(max_buffer_size)
        .invoke()?;

    Ok(())
}
```

### 2. Batch Operations with Merkle Proofs

```rust
pub fn batch_mint_compressed(
    ctx: Context<BatchMint>,
    metadata_list: Vec<MetadataArgs>,
) -> Result<()> {
    for (index, metadata) in metadata_list.iter().enumerate() {
        // Generate leaf hash
        let leaf_hash = hash_metadata(metadata)?;
        
        // Update Merkle tree
        ctx.accounts.merkle_tree.append_leaf(leaf_hash)?;
        
        msg!("Minted compressed NFT #{}", index);
    }
    Ok(())
}
```

---

## 🔄 Advanced CPI Patterns

### 1. Dynamic CPI with Multiple Programs

```rust
pub fn complex_multi_program_operation(
    ctx: Context<ComplexOperation>,
    operation_type: u8,
) -> Result<()> {
    match operation_type {
        1 => {
            // CPI to DEX program
            let cpi_ctx = CpiContext::new(
                ctx.accounts.dex_program.to_account_info(),
                DexSwap {
                    user: ctx.accounts.user.to_account_info(),
                    token_account_a: ctx.accounts.token_a.to_account_info(),
                    token_account_b: ctx.accounts.token_b.to_account_info(),
                }
            );
            dex_program::cpi::swap(cpi_ctx, amount)?;
        },
        2 => {
            // CPI to lending program
            let cpi_ctx = CpiContext::new(
                ctx.accounts.lending_program.to_account_info(),
                LendingDeposit {
                    user: ctx.accounts.user.to_account_info(),
                    reserve: ctx.accounts.reserve.to_account_info(),
                }
            );
            lending_program::cpi::deposit(cpi_ctx, amount)?;
        },
        _ => return Err(ErrorCode::InvalidOperation.into()),
    }
    Ok(())
}
```

### 2. CPI with Return Data

```rust
pub fn get_price_and_swap(ctx: Context<PriceAndSwap>) -> Result<()> {
    // CPI to oracle for price
    let cpi_ctx = CpiContext::new(
        ctx.accounts.oracle_program.to_account_info(),
        GetPrice {
            price_feed: ctx.accounts.price_feed.to_account_info(),
        }
    );
    oracle_program::cpi::get_price(cpi_ctx)?;
    
    // Read return data
    let return_data = solana_program::program::get_return_data()
        .ok_or(ErrorCode::NoReturnData)?;
    
    let price: u64 = u64::from_le_bytes(
        return_data.1[..8].try_into()
            .map_err(|_| ErrorCode::InvalidReturnData)?
    );
    
    // Use price for swap calculation
    let swap_amount = calculate_swap_amount(price)?;
    
    // Execute swap
    execute_swap(ctx, swap_amount)?;
    
    Ok(())
}
```

---

## 🔄 Account Versioning and Migration

### 1. Versioned Account Structure

```rust
#[account]
pub struct VersionedAccount {
    pub version: u8,
    pub data: VersionedData,
}

#[derive(AnchorSerialize, AnchorDeserialize)]
pub enum VersionedData {
    V1(AccountDataV1),
    V2(AccountDataV2),
    V3(AccountDataV3),
}

impl VersionedAccount {
    pub fn migrate_to_v2(&mut self) -> Result<()> {
        match &self.data {
            VersionedData::V1(v1_data) => {
                self.data = VersionedData::V2(AccountDataV2 {
                    // Migrate fields from V1 to V2
                    old_field: v1_data.old_field,
                    new_field: Default::default(),
                });
                self.version = 2;
            },
            _ => return Err(ErrorCode::InvalidVersion.into()),
        }
        Ok(())
    }
}
```

### 2. Backward Compatibility

```rust
pub fn handle_versioned_operation(
    ctx: Context<VersionedOperation>,
    operation_data: Vec<u8>,
) -> Result<()> {
    let account = &mut ctx.accounts.versioned_account;
    
    match account.version {
        1 => handle_v1_operation(account, operation_data)?,
        2 => handle_v2_operation(account, operation_data)?,
        3 => handle_v3_operation(account, operation_data)?,
        _ => {
            // Auto-migrate to latest version
            account.migrate_to_latest()?;
            handle_v3_operation(account, operation_data)?;
        }
    }
    
    Ok(())
}
```

---

## 🧪 Advanced Testing Patterns

### 1. Integration Testing with Multiple Programs

```rust
#[tokio::test]
async fn test_complex_defi_flow() {
    let mut program_test = ProgramTest::new(
        "my_defi_program",
        my_defi_program::id(),
        processor!(my_defi_program::entry),
    );
    
    // Add external programs
    program_test.add_program(
        "spl_token",
        spl_token::id(),
        None,
    );
    program_test.add_program(
        "serum_dex",
        serum_dex::id(),
        None,
    );
    
    let (mut banks_client, payer, recent_blockhash) = program_test.start().await;
    
    // Create complex test scenario
    let user_keypair = Keypair::new();
    let token_mint = create_mint(&mut banks_client, &payer, &user_keypair.pubkey()).await;
    
    // Test full DeFi flow
    let result = execute_defi_strategy(
        &mut banks_client,
        &payer,
        &user_keypair,
        &token_mint,
        recent_blockhash,
    ).await;
    
    assert!(result.is_ok());
}
```

### 2. Property-Based Testing

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_price_calculation_properties(
        base_amount in 1u64..1_000_000,
        price_multiplier in 1u64..1000,
        fee_bps in 0u64..1000,
    ) {
        let result = calculate_swap_output(base_amount, price_multiplier, fee_bps);
        
        // Property: output should always be less than input due to fees
        prop_assert!(result.unwrap() < base_amount);
        
        // Property: higher fees should result in lower output
        let result_higher_fee = calculate_swap_output(base_amount, price_multiplier, fee_bps + 100);
        prop_assert!(result_higher_fee.unwrap() < result.unwrap());
    }
}
```

---

## ⚡ Compute Unit Optimization

### 1. Compute Budget Management

```rust
use solana_program::compute_budget::{self, ComputeBudgetInstruction};

pub fn create_optimized_transaction() -> Transaction {
    let mut instructions = vec![
        // Set compute unit limit
        ComputeBudgetInstruction::set_compute_unit_limit(200_000),
        
        // Set compute unit price for priority
        ComputeBudgetInstruction::set_compute_unit_price(1000),
    ];
    
    // Add your program instructions
    instructions.push(your_program_instruction());
    
    Transaction::new_with_payer(&instructions, Some(&payer.pubkey()))
}
```

### 2. Efficient Account Loading

```rust
// Use references instead of cloning large data
pub fn process_large_account_efficiently(
    ctx: Context<ProcessLargeAccount>,
) -> Result<()> {
    let account_data = &ctx.accounts.large_account.data;
    
    // Process data in chunks to avoid compute limits
    for chunk in account_data.chunks(1000) {
        process_chunk(chunk)?;
    }
    
    Ok(())
}
```

---

## 🌐 Oracle Integration Patterns

### 1. Price Feed Integration

```rust
use chainlink_solana;

#[derive(Accounts)]
pub struct GetOraclePrice<'info> {
    #[account(
        address = chainlink_solana::price_feeds::SOL_USD @ ErrorCode::InvalidPriceFeed
    )]
    pub price_feed: AccountInfo<'info>,
}

pub fn use_oracle_price(ctx: Context<GetOraclePrice>) -> Result<()> {
    let price_feed = &ctx.accounts.price_feed;
    let price_data = chainlink_solana::get_price(price_feed)?;
    
    require!(
        price_data.timestamp > Clock::get()?.unix_timestamp - 300, // 5 minutes
        ErrorCode::StalePrice
    );
    
    let current_price = price_data.price;
    msg!("Current SOL/USD price: {}", current_price);
    
    // Use price for calculations
    process_with_price(current_price)?;
    
    Ok(())
}
```

### 2. Custom Oracle Implementation

```rust
#[account]
pub struct PriceOracle {
    pub authority: Pubkey,
    pub price: u64,
    pub last_updated: i64,
    pub confidence: u64,
}

pub fn update_oracle_price(
    ctx: Context<UpdateOracle>,
    new_price: u64,
    confidence: u64,
) -> Result<()> {
    let oracle = &mut ctx.accounts.oracle;
    
    require!(
        ctx.accounts.authority.key() == oracle.authority,
        ErrorCode::UnauthorizedOracle
    );
    
    oracle.price = new_price;
    oracle.confidence = confidence;
    oracle.last_updated = Clock::get()?.unix_timestamp;
    
    emit!(PriceUpdated {
        price: new_price,
        confidence,
        timestamp: oracle.last_updated,
    });
    
    Ok(())
}
```

---

## 🚀 Deployment and Upgrade Strategies

### 1. Safe Program Upgrades

```bash
# 1. Deploy to testnet first
solana program deploy --url testnet target/deploy/program.so

# 2. Test thoroughly on testnet
anchor test --provider.cluster testnet

# 3. Deploy to mainnet with buffer account
solana program write-buffer --url mainnet-beta target/deploy/program.so

# 4. Set buffer authority
solana program set-buffer-authority <buffer-address> --new-buffer-authority <upgrade-authority>

# 5. Deploy from buffer (allows for atomic upgrade)
solana program deploy --url mainnet-beta --buffer <buffer-address> --program-id <program-id>
```

### 2. Feature Flags for Gradual Rollout

```rust
#[account]
pub struct FeatureFlags {
    pub new_feature_enabled: bool,
    pub advanced_mode_enabled: bool,
    pub emergency_mode: bool,
}

pub fn process_with_feature_gates(
    ctx: Context<ProcessWithFeatures>,
    operation: Operation,
) -> Result<()> {
    let flags = &ctx.accounts.feature_flags;
    
    if flags.emergency_mode {
        return Err(ErrorCode::EmergencyMode.into());
    }
    
    match operation {
        Operation::NewFeature => {
            require!(
                flags.new_feature_enabled,
                ErrorCode::FeatureDisabled
            );
            execute_new_feature(ctx)?;
        },
        Operation::Standard => {
            execute_standard_operation(ctx)?;
        },
    }
    
    Ok(())
}
```

---

## 🔐 Advanced Security Patterns

### 1. Reentrancy Protection

```rust
#[account]
pub struct ReentrancyGuard {
    pub locked: bool,
}

pub fn protected_function(
    ctx: Context<ProtectedFunction>,
) -> Result<()> {
    let guard = &mut ctx.accounts.reentrancy_guard;
    
    require!(!guard.locked, ErrorCode::ReentrancyDetected);
    
    guard.locked = true;
    
    // Execute protected logic
    let result = execute_critical_operation(ctx);
    
    guard.locked = false;
    
    result
}
```

### 2. Access Control with Roles

```rust
#[account]
pub struct RoleBasedAccess {
    pub admin: Pubkey,
    pub operators: Vec<Pubkey>,
    pub users: Vec<Pubkey>,
}

pub fn role_protected_function(
    ctx: Context<RoleProtectedFunction>,
    required_role: Role,
) -> Result<()> {
    let access_control = &ctx.accounts.access_control;
    let signer = ctx.accounts.signer.key();
    
    let has_permission = match required_role {
        Role::Admin => access_control.admin == signer,
        Role::Operator => access_control.operators.contains(&signer),
        Role::User => access_control.users.contains(&signer),
    };
    
    require!(has_permission, ErrorCode::InsufficientPermissions);
    
    // Execute role-specific logic
    execute_role_based_operation(ctx, required_role)?;
    
    Ok(())
}
```

---

## 📊 Monitoring and Analytics

### 1. Custom Events and Metrics

```rust
#[event]
pub struct DetailedTransactionEvent {
    pub user: Pubkey,
    pub transaction_type: String,
    pub amount: u64,
    pub timestamp: i64,
    pub gas_used: u64,
    pub success: bool,
}

pub fn emit_detailed_metrics(
    ctx: Context<EmitMetrics>,
    transaction_type: String,
    amount: u64,
    gas_used: u64,
) -> Result<()> {
    emit!(DetailedTransactionEvent {
        user: ctx.accounts.user.key(),
        transaction_type,
        amount,
        timestamp: Clock::get()?.unix_timestamp,
        gas_used,
        success: true,
    });
    
    Ok(())
}
```

### 2. Health Check Endpoints

```rust
#[derive(Accounts)]
pub struct HealthCheck<'info> {
    pub system_state: Account<'info, SystemState>,
}

pub fn health_check(ctx: Context<HealthCheck>) -> Result<HealthStatus> {
    let state = &ctx.accounts.system_state;
    
    let status = HealthStatus {
        is_healthy: !state.emergency_mode,
        last_update: state.last_activity,
        active_users: state.active_user_count,
        system_load: calculate_system_load(state)?,
    };
    
    msg!("Health Status: {:?}", status);
    Ok(status)
}
```

---

## 🎯 What's Next?

Congratulations! You've completed the comprehensive Solana development course. You now have:

- **Fundamental Knowledge**: Understanding of Solana's architecture and development model
- **Practical Skills**: Hands-on experience with Anchor and raw Solana development
- **Advanced Techniques**: Knowledge of production-ready patterns and best practices
- **Security Awareness**: Understanding of common pitfalls and security considerations

### 🚀 Continue Your Journey:

1. **Build Real Projects**: Apply these patterns to create your own DeFi protocols, NFT marketplaces, or DAOs
2. **Contribute to Open Source**: Join the Solana ecosystem by contributing to existing projects
3. **Stay Updated**: Follow Solana development updates and new features
4. **Join the Community**: Engage with other developers in Discord, forums, and local meetups
5. **Teach Others**: Share your knowledge and help grow the Solana developer community

### 📚 Additional Resources:

- [Solana Cookbook](https://solanacookbook.com/)
- [Anchor Documentation](https://anchor-lang.com/)
- [Solana Program Library](https://spl.solana.com/)
- [Metaplex Documentation](https://docs.metaplex.com/)
- [Solana Security Best Practices](https://github.com/solana-labs/solana-program-library/blob/master/docs/security-audit.md)

Happy building on Solana! 🎉