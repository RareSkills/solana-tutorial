# Solana Instruction Introspection

Instruction introspection enables a Solana program to read an instruction other than its own within the same transaction.

Normally, a program can only read the instruction targeted to itself. The Solana runtime routes each instruction to the program specified in the instruction.

A Solana transaction can contain multiple instructions, each targeting a different program. For example, program **A** might receive instruction `Ax` and program **B** instruction `Bx` in the same transaction. Through introspection, program B can read the contents of both instruction `Ax` and `Bx`.

For example, suppose you want to ensure that any interaction with your DeFi program must be preceded with a transfer of `0.5` SOL to your treasury within the same transaction. You can enforce this rule by introspecting the instructions and rejecting the entire transaction if the required `0.5` SOL transfer instruction is not included prior to the instruction which interacts with your program.

In this article, we’ll learn how introspection works and how to implement it in your Solana program.

## Transaction and instructions

Before we take a look at instruction introspection, let’s review transactions and instructions in detail. 

A Solana transaction is a struct with two fields: a message and the signatures that signed it. The message contains an array of instructions to be executed sequentially. 

![A high-level diagram illustrating the components of a Solana transaction. It shows that a transaction consists of a "Message," which contains instructions, and an "Array of signatures" that signed the message.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image6.png)

The code below (which comes directly from the [Solana SDK](https://github.com/solana-labs/solana/blob/7700cb3128c1f19820de67b81aa45d18f73d2ac0/sdk/src/transaction/mod.rs#L173)) shows a struct representation of a transaction:

```rust
pub struct Transaction {
    pub signatures: Vec<Signature>,
    pub message: Message,
}
```

### The transaction message

The transaction message holds the list of instructions, and the union of all the account keys the instructions will collectively access. It also holds some additional data the runtime needs such as the recent block hash and the message header. 

```rust
pub struct Message {
    pub instructions: Vec<Instruction>,
    pub account_keys: Vec<Address>,
    pub recent_blockhash: Hash,
    pub header: MessageHeader,
}
```

Here’s a detailed breakdown of each component:

- **Instructions**: Each instruction is a call to an on-chain program. An instruction contains three components:
    - **Program ID**: The address of the program with the business logic for the instruction being called.
    - **Accounts:** Indexes into the transaction **account keys**. The indexes map the instruction to the specific accounts it needs to read from or write to.
    - **Instruction data:** A byte array specifying which function to call on the program and any arguments required by the instruction.
- **Account keys**: This is the union of all the accounts listed in the each of the instructions.
- **Recent blockhash**: A recent blockhash that ties the transaction to a short window of slots and prevents replay.
- **Message header**: This specifies how many accounts have signed the transaction, and which accounts are read-only versus writable.

**Instruction struct**

Below is the **Instruction** struct definition, from the [Solana source code on GitHub](https://github.com/solana-labs/solana/blob/7700cb3128c1f19820de67b81aa45d18f73d2ac0/sdk/program/src/instruction.rs#L633):

```rust
pub struct Instruction {
    /// Pubkey of the program that executes this instruction.
    pub program_id: Pubkey,
    /// Metadata describing accounts that should be passed to the program.
    pub accounts: Vec<AccountMeta>,
    /// Opaque data passed to the program for its own interpretation.
    pub data: Vec<u8>,
}

pub struct AccountMeta {
    /// An account's public key.
    pub pubkey: Pubkey,
    // True if the instruction requires a signature for this pubkey
    /// in the transaction's signatures list. 
    pub is_signer: bool,
    /// True if the account data or metadata may be mutated during program execution.
    pub is_writable: bool,
}
```

Each of the accounts an instruction uses is represented by the `AccountMeta` type, which stores the account’s public key along with the signer and writable flags.

**Summary of the relationship between transactions and instructions**

To put everything together, the image below shows the relationships between a transaction, a message, and instructions. 

A **Transaction** contains a list of signatures and a message. A **Message** contains a header, list of account keys, a recent blockhash, and a list of instructions. An **Instruction** contains a program ID, the accounts it uses (this indexes into the **account keys** list in the Message struct), and the instruction data. 

![A code snippet showing a nested Rust data structure for a Solana transaction. It demonstrates that the top-level Transaction struct is composed of signatures and a message. The Message struct contains a header, account_keys, the recent block_hash, and a vector of instructions. It also shows that the Instruction struct holds the program_id, accounts, and data.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image7.png)

## Instruction introspection with instruction Sysvar

Let’s discuss how introspection works by first examining the Solana Sysvar account.

A sysvar is a special read-only account that contains dynamically-updated data maintained by the Solana runtime and exposes internal network state to programs. We are literally reading the data from this account — we are not making a CPI to a program.

*We’ve discussed different types of Sysvars in a previous article in this series. To learn more about them, read the article “[Solana Sysvars Explained](https://rareskills.io/post/solana-sysvar)”.*

**Instruction introspection uses the instruction Sysvar account to access the serialized vector of instructions (program_id, accounts, and data) of the current transaction**. For example, in a transaction with multiple instructions, a program can read and analyze any of the instructions, not just the current instruction.

<video src="https://r2media.rareskills.io/SolanaInstructionIntrospection/video1.mp4" type="video/mp4" autoplay loop muted controls></video>
This animation shows an instruction introspection scenario where, while Instruction 1 is executing, the program can read the contents of Instruction 2 and Instruction 3.

**Unlike regular accounts in Solana, the instruction Sysvar account does not persist data; it is populated only for the lifetime of the transaction and cleared once execution completes.**

The instruction Sysvar account address is `Sysvar1nstructions1111111111111111111111111`. It contains the serialized list of all instructions in the current transaction. Each entry includes the program ID, accounts, and instruction data, like we saw earlier. Below is the Rust struct of each deserialized instruction, reproduced from earlier:

```rust
pub struct Instruction {
    /// Pubkey of the program that executes this instruction
    pub program_id: Pubkey,

    /// Metadata describing accounts that should be passed to the program
    pub accounts: Vec<AccountMeta>,

    /// Opaque data passed to the program for its own interpretation
    pub data: Vec<u8>,
}

```

The Solana Rust SDK provides several helper functions to access the serialized instructions in the instruction sysvar account. However, the SDK does not provide a single function that returns all instructions; instead, it only provides functions that deserialize a single instruction at a specific index. 

You can still read and deserialize the list of instructions in the sysvar account manually, but doing so is error-prone, hence, one should use the SDK for deserializing the instructions..

Here are the two key helper functions the Solana Rust SDK provides for introspection:

1. `load_current_index_checked` – Programs can use this helper function to learn their own index within the transaction list, then look up another instruction by their relative position.
2. `load_instruction_at_checked` – loads the instruction at a specific index and deserializes it into an `Instruction` struct. Once you have the current index using the `load_current_index_checked` function, you can use this function to introspect earlier or later instructions. We’ll see how to do this in a later section of this article.

First, to understand how these helper functions work, let’s look at the layout of the instruction sysvar account. It is organized into three regions:

1. The header
2. The instructions
3. And the index of the of the currently executing instruction

### **1. The header region**

The header specifies the number of instructions in the transaction and the instruction offsets (which point to where the instructions begin). The diagram below shows a header for a transaction with **2** instructions, so there are two offsets: one starting at memory location `6` and the other at memory location`20`.

 

![A diagram of a sysvar header's memory layout. The first 2 bytes represent num_instructions as a u16. This is followed by an array of u16 values representing the byte offsets to the start of each instruction, which are 6 and 20 in this example.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image1.png)

### 2. The instruction region

The instruction region begins at the byte position indicated by the offset (the red box in the diagram below is only a visual marker for the offset, not an actual memory location). From that position, it contains the account metadata, the program ID, the length of the instruction data, and finally the instruction data itself. If we have more than one instruction, this structure repeats for each one.

![A diagram showing an example layout of the instruction region from a Solana Instructions Sysvar account. It uses a red box to visually indicate the location an offset points to, marking the beginning of the instruction. The layout then shows the account information field (which contains num_accounts, account.meta, and account.pubkey), followed by the program_id, the data_len, and the data itself.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image2.png)

### 3. The index of the of the currently executing instruction

And finally, the index of the currently executing instruction is stored at the end of the Sysvar layout.

![A diagram showing the header, instructions, and the current instruction index region of the instruction sysvar account.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image3.png)

If the program knows the index of the currently executing instruction, it can get the other instructions relative to it.

## Accessing instructions

Now that we’ve looked into how data is laid out in the the Sysvar account, let’s look at a practical example. We’ll use the two helper methods for introspection: `load_current_index_checked` and `load_instruction_at_checked`, to access instructions in a transaction. For the purpose of this article, we’ll use a basic transfer transaction.

Our example program will verify that a system transfer instruction precedes its own instruction. The transaction will succeed only if this condition is met.

```rust
Transaction:
├── Instruction 0: System Transfer (user pays X lamports)
└── Instruction 1: This program (verifies the payment)
```

### Setting up the program

To follow along, you should have a Solana development environment setup, if you haven’t, read [our first article in this series](https://rareskills.io/post/hello-world-solana). 

Initialize a new Anchor application:

```bash
anchor init instruction-introspection
```

Update your dependency in `program/src/Cargo.toml` to include **bincode (**`bincode=1.3.3`**)**. We’ll use the **bincode** library to deserialize the system instruction: 

```bash
//... rest of toml file content

[dependencies]
anchor-lang = "0.31.1"
**bincode = "1.3.3" # add this**
```

We’ll use Devnet for this project. Create a `.env` file in your root directory and add the provider and wallet exports below:

```rust
export ANCHOR_PROVIDER_URL=https://api.devnet.solana.com
export ANCHOR_WALLET=~/.config/solana/id.json
```

Update the Anchor.toml file as well to use the devnet provider and wallet.

```rust
[provider]
cluster = "https://api.devnet.solana.com"
wallet = "~/.config/solana/id.json"
```

Also, because you’ll need some SOL to pay for fees on Devnet, run `solana airdrop 2` to get 2 SOL which will be more than enough for this example.

### **Imports**

Now, we’ll import the the Anchor dependencies we’ll use for this example to replace the code in `program/src/lib.rs` file. Importantly, we import the `load_instruction_at_checked` and the `load_current_index_checked` from `sysvar::instructions`:

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{    
         system_program,    
         sysvar::instructions::{
                load_instruction_at_checked, 
                load_current_index_checked
         },    
         system_instruction::SystemInstruction,
};
```

And then we’ll declare the program ID and add a `verify_transfer` function, which will:

1. Get the current instruction index to understand the position of the currently executing transaction.
2. Load the previous instruction by deserializing the list of instructions in the sysvar account using the on-chain Solana Rust SDK.
3. Verify the loaded instruction is a system transfer instruction by checking the program ID matches the system program, then parse the instruction data to confirm the transfer amount matches the expected amount
4. Verify the number of accounts involved in the instruction is 2
5. And finally, we’ll define the struct for the sysvar account.

See the full code below. We've added comments to annotate the steps listed above:

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    system_program,
    sysvar::instructions::{load_instruction_at_checked, load_current_index_checked},
    system_instruction::SystemInstruction,
};

declare_id!("BxQuawTcvJkT2JM1qKeW6wyM4i5VCuM122v9tVsSrmwm");

#[program]
pub mod check_transfer {
    use super::*;

    pub fn verify_transfer(ctx: Context<VerifyTransfer>, expected_amount: u64) -> Result<()> {
        // Step 1: Get current instruction index to understand our position
        **let current_ix_index = load_current_index_checked(&ctx.accounts.instruction_sysvar)?;**
        msg!("Currently executing instruction index: {}", current_ix_index);

        // Step 2: Load the previous instruction
        let transfer_ix = load_instruction_at_checked(
            (current_ix_index - 1) as usize,
            &ctx.accounts.instruction_sysvar
        ).map_err(|_| error!(ErrorCode::MissingInstruction))?;

        // Step 3:  Verify it's a system program instruction
        require_keys_eq!(transfer_ix.program_id, system_program::ID, ErrorCode::NotSystemProgram);

        // Step 4: Parse the system instruction data
        let system_ix = bincode::deserialize(&transfer_ix.data)
            .map_err(|_| error!(ErrorCode::InvalidInstructionData))?;

        match system_ix {
            SystemInstruction::Transfer { lamports } => {
                require_eq!(lamports, expected_amount, ErrorCode::IncorrectAmount);
                msg!("✅ Verified transfer of {} lamports", lamports);
            }
            _ => return Err(error!(ErrorCode::NotTransferInstruction)),
        }

        // Step 5: Verify accounts involved in the transfer
        require_gte!(transfer_ix.accounts.len(), 2, ErrorCode::InsufficientAccounts);
        
        let from_account = &transfer_ix.accounts[0];
        let to_account = &transfer_ix.accounts[1];
        
        require!(from_account.is_signer, ErrorCode::FromAccountNotSigner);
        require!(from_account.is_writable, ErrorCode::FromAccountNotWritable);
        require!(to_account.is_writable, ErrorCode::ToAccountNotWritable);

        msg!("✅ Transfer accounts properly configured");
        msg!("From: {}", from_account.pubkey);
        msg!("To: {}", to_account.pubkey);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct VerifyTransfer<'info> {
    /// CHECK: This is the instruction sysvar account
    #[account(address = anchor_lang::solana_program::sysvar::instructions::ID)]
    pub instruction_sysvar: AccountInfo<'info>,
}

```

Here are the error codes we used, you should add to the same file:

```rust
#[error_code]
pub enum ErrorCode {
    /// Thrown when attempting to load an instruction at an index that doesn't exist
    /// in the transaction (e.g., trying to access index -1 when current is 0)
    #[msg("Missing required instruction in transaction")]
    MissingInstruction,
    
    /// Thrown when the previous instruction's program_id doesn't match the System Program
    /// Ensures we're only validating actual system program instructions
    #[msg("Instruction is not from System Program")]
    NotSystemProgram,
    
    /// Thrown when bincode fails to deserialize the instruction data into SystemInstruction
    /// Indicates malformed or corrupted instruction data
    #[msg("Invalid instruction data format")]
    InvalidInstructionData,
    
    /// Thrown when the SystemInstruction variant is not Transfer
    /// (e.g., it's CreateAccount, Allocate, or another system instruction type)
    #[msg("Instruction is not a transfer")]
    NotTransferInstruction,
    
    /// Thrown when the actual lamports amount in the transfer doesn't equal expected_amount
    /// Protects against front-running or incorrect payment amounts
    #[msg("Transfer amount does not match expected amount")]
    IncorrectAmount,
    
    /// Thrown when the transfer instruction has fewer than 2 accounts
    /// A valid transfer requires at least [from, to] accounts
    #[msg("Transfer instruction has insufficient accounts")]
    InsufficientAccounts,
    
    /// Thrown when the 'from' account in the transfer didn't sign the transaction
    /// Prevents unauthorized transfers
    #[msg("From account is not a signer")]
    FromAccountNotSigner,
    
    /// Thrown when the 'from' account is not marked as writable
    /// Required because the account balance will be debited
    #[msg("From account is not writable")]
    FromAccountNotWritable,
    
    /// Thrown when the 'to' account is not marked as writable
    /// Required because the account balance will be credited
    #[msg("To account is not writable")]
    ToAccountNotWritable,
}
```

In the above code, we got our current instruction index, used the ID to load the previous instruction for inspection. We are able to load that by just subtracting the current index by 1 since the instructions are in a sequential order.

Now, let’s build, deploy the program, and interact with it using JavaScript.

Run `anchor build && anchor deploy`  to build and deploy the project. You should see an output like this to show that it was successfully deployed:

![image.png](https://r2media.rareskills.io/SolanaInstructionIntrospection/image4.png)

### Interacting with the program code using Typescript

Create a simple Typescript script to transfer 1 SOL to an address with our program. 

To run the Typescript files directly, you'll use bun.js. If you don't already have it installed, you can install it by running `curl -fsSL [https://bun.sh/install](https://bun.sh/install) | bash` on your terminal.

Create a **`scripts/`** folder, add an `introspect.ts` file, and paste the code below in it. I've added  comments to help you understand the flow of ideas in the code.

```rust
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SystemProgram, SYSVAR_INSTRUCTIONS_PUBKEY, Transaction, Keypair } from "@solana/web3.js";
import { CheckTransfer } from "../target/types/check_transfer";

async function main() {
  console.log("🚀 Starting verification script...");

  // --- Setup Connection and Program ---
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  // Load the Anchor program from the workspace.
  const program = anchor.workspace.CheckTransfer as Program<CheckTransfer>;

  // --- Prepare Accounts and Data ---
  // The 'payer' is the wallet that signs and pays for the transaction.
  const payer = provider.wallet.publicKey;
  // A new, random keypair to act as the recipient.
  const recipient = Keypair.generate().publicKey;

  // Define the transfer amount using anchor.BN for u64 safety.
  const transferAmount = new anchor.BN(1_000_000_000); // 1 SOL

  console.log(`- Payer: ${payer}`);
  console.log(`- Recipient: ${recipient}`);
  console.log(`- Amount: ${transferAmount.toString()} lamports`);

  // --- Build the Transaction ---
  // A transaction is a container for one or more instructions.
  const tx = new Transaction();

  // Instruction 0: The System Program Transfer.
  // This must immediately precede our program's instruction.
  tx.add(
    SystemProgram.transfer({
      fromPubkey: payer,
      toPubkey: recipient,
      lamports: transferAmount.toNumber(), // Safe for 1 SOL
    })
  );

  // Instruction 1: Our program's verification instruction.
  tx.add(
    await program.methods
      .verifyTransfer(transferAmount)
      .accounts({
        instructionSysvar: SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction()
  );

  // --- Send Transaction and Verify Outcome ---
  try {
    const sig = await provider.sendAndConfirm(tx);
    console.log("\n✅ Transaction confirmed!");
    console.log(`Signature: ${sig}`);

    // Fetch the transaction details to inspect the logs.
    const txInfo = await provider.connection.getTransaction(sig, {
      commitment: "confirmed",
      maxSupportedTransactionVersion: 0,
    });

    console.log("\n📄 Program Logs:");
    console.log(txInfo?.meta?.logMessages?.join("\n"));

    // Check for the success message in the logs.
    const logs = txInfo?.meta?.logMessages;
    if (!logs || !logs.some(log => log.includes(`Verified transfer of ${transferAmount} lamports`))) {
        throw new Error("Verification log message not found!");
    }
    console.log("\n✅ Verification successful!");

  } catch (error) {
    console.error("\n❌ Transaction failed!");
    console.error(error);
    process.exit(1); // Exit with a non-zero error code
  }
}

// --- Script Entrypoint ---
main().then(
  () => process.exit(0),
  err => {
    console.error(err);
    process.exit(1);
  }
);

```

When we run the client code with `bun run script/introspect.ts`, we should see that it works with an output like this one:

![A screenshot of a terminal showing the result of a Solana introspection test. It confirms that the introspection was successful.](https://r2media.rareskills.io/SolanaInstructionIntrospection/image5.png)

### **Precaution for instruction introspection: a**void using absolute indexes during inspection

Loading an instruction from an absolute index like `0` from the sysvar account can allow an attacker to reuse that instruction across multiple calls.

For example, if your program requires a user to transfer funds to your treasury before withdrawing in the same transaction, using an absolute index could let an attacker place a single transfer at index `0` and then make multiple withdrawals that all validate against that same transfer.

Instead, use relative instruction indexing to ensure the transfer occurs immediately before the withdrawal instruction like we showed earlier in our example.

```rust
 let transfer_ix = load_instruction_at_checked(
    (current_ix_index - 1) as usize,
     &ctx.accounts.instruction_sysvar
     ) 
```

This ensures the inspected instruction is the correct transfer for the current withdrawal, not a reused transfer from earlier in the transaction.

*This article is part of a [tutorial series on Solana](https://rareskills.io/solana-tutorial).*