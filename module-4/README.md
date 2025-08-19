# 🌐 Module 4: The No-BS Web3 — Client Calls That Actually Work

**Goal:** Build a TypeScript client using raw `@solana/web3.js` that calls our escrow program without Anchor's helper methods. Understand exactly what's happening under the hood.

---

## 🎯 What You'll Learn

- ✅ Generate PDAs on the client side
- ✅ Build raw transactions using `@solana/web3.js`
- ✅ Understand instruction data layout and encoding
- ✅ Call instructions **manually** (no Anchor client helper)
- ✅ Decode program logs and transaction results

---

## 🚀 Raw Web3.js Implementation

### 📁 Project Setup

```bash
# Create new Node.js project
mkdir escrow-raw-client
cd escrow-raw-client
npm init -y

# Install dependencies
npm install @solana/web3.js @solana/spl-token borsh
npm install -D typescript ts-node @types/node
```

### 🔧 TypeScript Configuration (`tsconfig.json`)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 🏗️ Raw Client Implementation (`src/escrow-client.ts`)

```typescript
import {
  Connection,
  PublicKey,
  Keypair,
  Transaction,
  TransactionInstruction,
  SystemProgram,
  SYSVAR_RENT_PUBKEY,
  sendAndConfirmTransaction,
  AccountMeta,
} from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  createMint,
  createAccount,
  mintTo,
  getAccount,
} from "@solana/spl-token";
import { serialize, deserialize } from "borsh";

// Program ID from our Anchor program
const PROGRAM_ID = new PublicKey("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

// Instruction discriminators (8-byte hashes of instruction names)
const INSTRUCTION_DISCRIMINATORS = {
  initialize_escrow: Buffer.from([175, 175, 109, 31, 13, 152, 155, 237]),
  complete_escrow: Buffer.from([163, 52, 200, 231, 140, 3, 69, 186]), 
  cancel_escrow: Buffer.from([181, 18, 70, 0, 49, 89, 253, 124]),
};

// Borsh schemas for instruction data
class InitializeEscrowArgs {
  amount: bigint;
  
  constructor(props: { amount: bigint }) {
    this.amount = props.amount;
  }
}

const INITIALIZE_ESCROW_SCHEMA = new Map([
  [InitializeEscrowArgs, { kind: 'struct', fields: [['amount', 'u64']] }]
]);

// Escrow state account structure
class EscrowState {
  initializer: PublicKey;
  mint: PublicKey;
  amount: bigint;
  vault_authority_bump: number;

  constructor(props: {
    initializer: PublicKey;
    mint: PublicKey;
    amount: bigint;
    vault_authority_bump: number;
  }) {
    this.initializer = props.initializer;
    this.mint = props.mint;
    this.amount = props.amount;
    this.vault_authority_bump = props.vault_authority_bump;
  }
}

const ESCROW_STATE_SCHEMA = new Map([
  [EscrowState, {
    kind: 'struct',
    fields: [
      ['initializer', [32]], // PublicKey as 32-byte array
      ['mint', [32]],
      ['amount', 'u64'],
      ['vault_authority_bump', 'u8'],
    ]
  }]
]);

export class RawEscrowClient {
  constructor(
    private connection: Connection,
    private programId: PublicKey = PROGRAM_ID
  ) {}

  // Derive PDA addresses
  async findEscrowState(
    initializer: PublicKey,
    mint: PublicKey
  ): Promise<[PublicKey, number]> {
    return PublicKey.findProgramAddressSync(
      [
        Buffer.from("escrow"),
        initializer.toBuffer(),
        mint.toBuffer(),
      ],
      this.programId
    );
  }

  async findVaultAuthority(escrowState: PublicKey): Promise<[PublicKey, number]> {
    return PublicKey.findProgramAddressSync(
      [
        Buffer.from("vault-authority"),
        escrowState.toBuffer(),
      ],
      this.programId
    );
  }

  async findVaultTokenAccount(escrowState: PublicKey): Promise<[PublicKey, number]> {
    return PublicKey.findProgramAddressSync(
      [
        Buffer.from("vault"),
        escrowState.toBuffer(),
      ],
      this.programId
    );
  }

  // Create initialize escrow instruction
  createInitializeEscrowInstruction(
    initializer: PublicKey,
    mint: PublicKey,
    initializerTokenAccount: PublicKey,
    escrowState: PublicKey,
    vaultAuthority: PublicKey,
    vaultTokenAccount: PublicKey,
    amount: bigint
  ): TransactionInstruction {
    // Serialize instruction data
    const args = new InitializeEscrowArgs({ amount });
    const data = Buffer.concat([
      INSTRUCTION_DISCRIMINATORS.initialize_escrow,
      Buffer.from(serialize(INITIALIZE_ESCROW_SCHEMA, args))
    ]);

    const keys: AccountMeta[] = [
      { pubkey: initializer, isSigner: true, isWritable: true },
      { pubkey: mint, isSigner: false, isWritable: false },
      { pubkey: initializerTokenAccount, isSigner: false, isWritable: true },
      { pubkey: escrowState, isSigner: false, isWritable: true },
      { pubkey: vaultAuthority, isSigner: false, isWritable: false },
      { pubkey: vaultTokenAccount, isSigner: false, isWritable: true },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: SYSVAR_RENT_PUBKEY, isSigner: false, isWritable: false },
    ];

    return new TransactionInstruction({
      keys,
      programId: this.programId,
      data,
    });
  }

  // Create complete escrow instruction
  createCompleteEscrowInstruction(
    receiver: PublicKey,
    initializer: PublicKey,
    escrowState: PublicKey,
    vaultAuthority: PublicKey,
    vaultTokenAccount: PublicKey,
    receiverTokenAccount: PublicKey
  ): TransactionInstruction {
    const data = INSTRUCTION_DISCRIMINATORS.complete_escrow;

    const keys: AccountMeta[] = [
      { pubkey: receiver, isSigner: true, isWritable: true },
      { pubkey: initializer, isSigner: false, isWritable: true },
      { pubkey: escrowState, isSigner: false, isWritable: true },
      { pubkey: vaultAuthority, isSigner: false, isWritable: false },
      { pubkey: vaultTokenAccount, isSigner: false, isWritable: true },
      { pubkey: receiverTokenAccount, isSigner: false, isWritable: true },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
    ];

    return new TransactionInstruction({
      keys,
      programId: this.programId,
      data,
    });
  }

  // Create cancel escrow instruction  
  createCancelEscrowInstruction(
    initializer: PublicKey,
    escrowState: PublicKey,
    vaultAuthority: PublicKey,
    vaultTokenAccount: PublicKey,
    initializerTokenAccount: PublicKey
  ): TransactionInstruction {
    const data = INSTRUCTION_DISCRIMINATORS.cancel_escrow;

    const keys: AccountMeta[] = [
      { pubkey: initializer, isSigner: true, isWritable: true },
      { pubkey: escrowState, isSigner: false, isWritable: true },
      { pubkey: vaultAuthority, isSigner: false, isWritable: false },
      { pubkey: vaultTokenAccount, isSigner: false, isWritable: true },
      { pubkey: initializerTokenAccount, isSigner: false, isWritable: true },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
    ];

    return new TransactionInstruction({
      keys,
      programId: this.programId,
      data,
    });
  }

  // Fetch and decode escrow state
  async getEscrowState(escrowStatePubkey: PublicKey): Promise<EscrowState | null> {
    try {
      const accountInfo = await this.connection.getAccountInfo(escrowStatePubkey);
      if (!accountInfo || !accountInfo.data) {
        return null;
      }

      // Skip 8-byte discriminator
      const data = accountInfo.data.slice(8);
      return deserialize(ESCROW_STATE_SCHEMA, EscrowState, data);
    } catch (error) {
      console.error("Error fetching escrow state:", error);
      return null;
    }
  }
}
```

### 🧪 Complete Usage Example (`src/example.ts`)

```typescript
import { Connection, Keypair, LAMPORTS_PER_SOL, clusterApiUrl } from "@solana/web3.js";
import {
  TOKEN_PROGRAM_ID,
  createMint,
  createAccount,
  mintTo,
  getAccount,
} from "@solana/spl-token";
import { RawEscrowClient } from "./escrow-client";

async function main() {
  console.log("🚀 Starting raw escrow client example...");

  // Connect to localnet
  const connection = new Connection("http://127.0.0.1:8899", "confirmed");
  const client = new RawEscrowClient(connection);

  // Create test accounts
  const initializer = Keypair.generate();
  const receiver = Keypair.generate();

  console.log("💰 Airdropping SOL...");
  await Promise.all([
    connection.requestAirdrop(initializer.publicKey, 2 * LAMPORTS_PER_SOL),
    connection.requestAirdrop(receiver.publicKey, 2 * LAMPORTS_PER_SOL),
  ]);

  // Wait for airdrops to confirm
  await new Promise(resolve => setTimeout(resolve, 2000));

  // Create mint
  console.log("🪙 Creating mint...");
  const mint = await createMint(
    connection,
    initializer,
    initializer.publicKey,
    null,
    6
  );

  // Create token accounts
  console.log("📄 Creating token accounts...");
  const initializerTokenAccount = await createAccount(
    connection,
    initializer,
    mint,
    initializer.publicKey
  );

  const receiverTokenAccount = await createAccount(
    connection,
    receiver,
    mint,
    receiver.publicKey
  );

  // Mint tokens to initializer
  console.log("🏭 Minting tokens...");
  await mintTo(
    connection,
    initializer,
    mint,
    initializerTokenAccount,
    initializer,
    2000 * 10**6 // 2000 tokens
  );

  // Derive PDAs
  console.log("🔍 Deriving PDAs...");
  const [escrowState] = await client.findEscrowState(initializer.publicKey, mint);
  const [vaultAuthority] = await client.findVaultAuthority(escrowState);
  const [vaultTokenAccount] = await client.findVaultTokenAccount(escrowState);

  console.log("PDAs:");
  console.log("- Escrow State:", escrowState.toString());
  console.log("- Vault Authority:", vaultAuthority.toString());
  console.log("- Vault Token Account:", vaultTokenAccount.toString());

  // Initialize escrow
  console.log("🔄 Initializing escrow...");
  const escrowAmount = BigInt(1000 * 10**6); // 1000 tokens

  const initInstruction = client.createInitializeEscrowInstruction(
    initializer.publicKey,
    mint,
    initializerTokenAccount,
    escrowState,
    vaultAuthority,
    vaultTokenAccount,
    escrowAmount
  );

  const initTx = new Transaction().add(initInstruction);
  const initSignature = await sendAndConfirmTransaction(
    connection,
    initTx,
    [initializer],
    { commitment: "confirmed" }
  );

  console.log("✅ Escrow initialized! Signature:", initSignature);

  // Verify escrow state
  const escrowStateData = await client.getEscrowState(escrowState);
  console.log("📊 Escrow state:", {
    initializer: escrowStateData?.initializer.toString(),
    amount: escrowStateData?.amount.toString(),
    mint: escrowStateData?.mint.toString(),
  });

  // Verify vault has tokens
  const vaultAccount = await getAccount(connection, vaultTokenAccount);
  console.log("🏦 Vault balance:", vaultAccount.amount.toString());

  // Complete escrow
  console.log("🔄 Completing escrow...");
  const completeInstruction = client.createCompleteEscrowInstruction(
    receiver.publicKey,
    initializer.publicKey,
    escrowState,
    vaultAuthority,
    vaultTokenAccount,
    receiverTokenAccount
  );

  const completeTx = new Transaction().add(completeInstruction);
  const completeSignature = await sendAndConfirmTransaction(
    connection,
    completeTx,
    [receiver],
    { commitment: "confirmed" }
  );

  console.log("✅ Escrow completed! Signature:", completeSignature);

  // Verify receiver got tokens
  const receiverAccount = await getAccount(connection, receiverTokenAccount);
  console.log("💎 Receiver final balance:", receiverAccount.amount.toString());

  // Verify escrow state is closed
  const finalEscrowState = await client.getEscrowState(escrowState);
  console.log("🗑️ Escrow state closed:", finalEscrowState === null);

  console.log("🎉 Raw client example completed successfully!");
}

main().catch(console.error);
```

### 📦 Package Scripts (`package.json`)

```json
{
  "scripts": {
    "build": "tsc",
    "start": "ts-node src/example.ts",
    "dev": "ts-node --watch src/example.ts"
  }
}
```

---

## 🧠 Key Learning Points

**1. Instruction Discriminators**
```typescript
// These are 8-byte hashes of "global:initialize_escrow" etc.
const INSTRUCTION_DISCRIMINATORS = {
  initialize_escrow: Buffer.from([175, 175, 109, 31, 13, 152, 155, 237]),
  // ... computed by Anchor, but you can generate them manually
};
```

**2. Manual PDA Derivation**
```typescript
const [escrowState] = PublicKey.findProgramAddressSync(
  [Buffer.from("escrow"), initializer.toBuffer(), mint.toBuffer()],
  programId
);
```

**3. Raw Transaction Building**
```typescript
const keys: AccountMeta[] = [
  { pubkey: initializer, isSigner: true, isWritable: true },
  // ... exact order and permissions matter
];
```

**4. Borsh Serialization**
```typescript
const args = new InitializeEscrowArgs({ amount });
const data = Buffer.concat([
  discriminator,
  Buffer.from(serialize(schema, args))
]);
```

**5. Account Data Deserialization**
```typescript
// Skip 8-byte discriminator, then deserialize
const data = accountInfo.data.slice(8);
return deserialize(ESCROW_STATE_SCHEMA, EscrowState, data);
```

---

## 🚀 Running the Example

```bash
# Start local validator (in another terminal)
solana-test-validator

# Build and run
npm run build
npm start
```

### 📊 Expected Output

```text
🚀 Starting raw escrow client example...
💰 Airdropping SOL...
🪙 Creating mint...
📄 Creating token accounts...
🏭 Minting tokens...
🔍 Deriving PDAs...
PDAs:
- Escrow State: 8x7AbC...def
- Vault Authority: 9y8BdE...fgh  
- Vault Token Account: Az9CeF...ghi
🔄 Initializing escrow...
✅ Escrow initialized! Signature: 5x6Y...789
📊 Escrow state: {
  initializer: 'ABC...123',
  amount: '1000000000',
  mint: 'DEF...456'
}
🏦 Vault balance: 1000000000
🔄 Completing escrow...
✅ Escrow completed! Signature: 6z7X...890
💎 Receiver final balance: 1000000000
🗑️ Escrow state closed: true
🎉 Raw client example completed successfully!
```

---

## 🔍 Under the Hood Comparison

| **Anchor SDK** | **Raw Web3.js** | **What We Learned** |
|----------------|-----------------|---------------------|
| `program.methods.initializeEscrow()` | `new TransactionInstruction()` | Manual instruction building |
| Auto PDA derivation | `PublicKey.findProgramAddressSync()` | PDA seeds and program ownership |
| Auto serialization | `serialize(schema, data)` | Borsh encoding/decoding |
| IDL type safety | Manual schema definition | Account layout structure |
| `program.account.fetch()` | `getAccountInfo() + deserialize()` | Raw account data parsing |

---

## 🎯 What's Next?

In [Module 5](../module-5/README.md), we'll use Anchor's `banks-client` to simulate mainnet conditions locally, importing real accounts for realistic testing.