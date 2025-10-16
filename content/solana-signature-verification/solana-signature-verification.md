# Ed25519 Signature Verification in Solana

# **Verifying Ed25519 Signatures in Solana Anchor Programs**

![Frame 4.png](https://www.notion.so/images/app-packages/docs-getting-started-1.png)

This tutorial shows how to verify an off-chain Ed25519 signature in a Solana program.

In Solana, custom programs generally don't implement cryptographic primitives like `Ed25519` or `Secp256k1` signature verification themselves because such operations are compute-intensive and would consume excessive [compute units](https://rareskills.io/post/solana-compute-unit-price) in the SVM.

Instead, Solana provides `Ed25519Program` and `Secp256k1Program` as native programs that are optimized for signature verification. This is similar to how Ethereum uses a [precompile](https://rareskills.io/post/solidity-precompiles) to validate ECDSA signatures because implementing that logic directly in EVM bytecode would be too gas-intensive.

Although wallet transactions are also signed with `Ed25519`, those signatures are verified by the Solana runtime itself, not the `Ed25519Program`. The `Ed25519Program` is used when you need to verify signatures included inside transaction instruction data, such as a distributor’s signature for an airdrop claim.

In this article, we’ll show how signature verification works in Solana using `Ed25519Program` and [instruction introspection](https://rareskills.io/post/solana-instruction-introspection). Our running example will be an airdrop flow, where a distributor signs claims off-chain and recipients submit those signed claims on-chain for verification so they can claim the airdrop.

## Ed25519Program is stateless

The Solana Ed25519Program only performs cryptographic signature verification based on the provided input parameters. It doesn't maintain any persistent data between calls, therefore, it owns no accounts. As a result, it doesn’t store the outcome of verification. If the signature verification fails, the entire transaction is rejected; if it succeeds, execution continues and the next instruction can safely assume the signature was valid.

## Our running example: Airdrop

In an airdrop, we need a way to know who is eligible to claim tokens. One approach is to store all eligible addresses on-chain, but this is costly.

Rather than storing all recipient addresses on-chain, a signature-based airdrop uses a trusted distributor (e.g. the project’s team) to sign off-chain messages containing each recipient’s wallet address and token amount `(recipient, amount)`. The on-chain program responsible for distributing the airdrop verifies these signatures to authorize token claims and transfer the `amount` to the `recipient`.

## How the verification process works

The signature verification process uses instruction introspection, where a program can read other instructions in the same transaction. We discussed instruction introspection previously, and now we'll focus on how it applies to signature verification.

First, our airdrop recipient submits a single transaction with two instructions, we’ll refer to them as **`Ed25519 Instruction`** for instruction 1 and **`AirdropClaim Instruction`** for instruction 2 in this article:

Recall that an instruction contains a program ID, a list of accounts, and arbitrary data that the program interprets. We’ll make reference to this instruction struct throughout this article:

```rust
pub struct Instruction {
    /// Pubkey of the program that executes this instruction.
    pub program_id: Pubkey,
    /// Metadata describing accounts that should be passed to the program.
    pub accounts: Vec<AccountMeta>,
    /// Opaque data passed to the program for its own interpretation.
    pub data: Vec<u8>,
}
```

## **Instruction 1: `Ed25519 Instruction` for** signature verification

The `Ed25519 Instruction` is a Solana instruction whose `program_id` is the native `Ed25519Program` verifier (`Ed25519SigVerify111111111111111111111111111`). It is the first instruction in our airdrop transaction.

Since the `Ed25519Program` is stateless, no accounts are needed for this instruction, so all inputs are encoded in the instruction `data`.

### How instruction data for `Ed25519Program` is formatted

The `data`  in the `Ed25519program` instruction starts with a 16-byte header which contains the number of signatures in the instruction and offsets. In our case, we’ll have only the distributor’s signature count and the offsets. These offsets point into the rest of the `data` to locate the public key, message, and signature that were verified. The rest of the data will continue from the 16th byte to the 151st byte.

|  |  | **Ed25519 Instruction** |  |
| --- | --- | --- | --- |
|    [bytes 0..15] 

Header (16 bytes) |        [bytes 16..47]

Distributor’s public key (32 bytes) |       [bytes 48..111]

Distributor’s Signature (64 bytes) |              [bytes 112..151]

Message
   - Recipient Pubkey (0..31) 
   -  Amount of airdrop 
       token(32..39, little-endian) |

This is the Rust struct of the header:

```rust
struct Ed25519InstructionHeader {
    num_signatures: u8,   // 1 byte
    padding: u8,          // 1 byte
    offsets: Ed25519SignatureOffsets, // 14 bytes
}

struct Ed25519SignatureOffsets {
    signature_offset: u16,             // 2 bytes
    signature_instruction_index: u16,  // 2 bytes
    public_key_offset: u16,            // 2 bytes
    public_key_instruction_index: u16, // 2 bytes
    message_data_offset: u16,          // 2 bytes
    message_data_size: u16,            // 2 bytes
    message_instruction_index: u16,    // 2 bytes
}
```

Notice that  the `Ed25519SignatureOffsets` struct has the following indices: `signature_instruction_index`, `public_key_instruction_index`, and `message_instruction_index`. These indices are used to determine if the instruction data is in the current instruction being executed. The indices in the current instruction data are set to `u16::MAX` in the Solana [Ed25519 source code](https://github.com/anza-xyz/solana-sdk/blob/c654e5f556ad3e22679fe9757da1bf5c9486e2f1/ed25519-program/src/lib.rs#L79): 

```rust
    let offsets = Ed25519SignatureOffsets {
        signature_offset: signature_offset as u16,
        signature_instruction_index: u16::MAX,
        public_key_offset: public_key_offset as u16,
        public_key_instruction_index: u16::MAX,
        message_data_offset: message_data_offset as u16,
        message_data_size: message.len() as u16,
        message_instruction_index: u16::MAX,
    };
```

Any other value would point to another instruction in the transaction.

The layout for the **`Ed25519 Instruction`** data will look like this in our running airdrop example. 

|  |  | **Ed25519 Instruction** |  |
| --- | --- | --- | --- |
|          0..15
Header (16 bytes) |              16..47
Distributor’s public key |            48..111
Distributor’s Signature |           112..151
Message
   - Recipient Pubkey (0..31) 
   -  Amount of airdrop 
       token(32..39, little-endian) |

In practice, you’ll use off-chain helpers like Web3.js or the [solana-ed25519-program](https://crates.io/crates/solana-ed25519-program) crate to build a valid instruction. Below is a snippet from the ed25519 crate source code showing the input parameters to build the instruction and then return a valid instruction off-chain. (The Typescript version will be shown later)

```rust
use solana_ed25519_program::new_ed25519_instruction_with_signature;

pub fn new_ed25519_instruction_with_signature(
    message: &[u8],
    signature: &[u8; 64],
    pubkey: &[u8; 32],
) -> Instruction
```

Conceptually, the de-serialized version of `Ed25519 Instruction` looks like this:

|  | Ed25519 Instruction |
| --- | --- |
| Program ID | Ed25519SigVerify111111111111111111111111111 |
| Accounts | [] |
| Instruction Data |     - Header (Signature Count + Offsets) 
    - Distributor's Public Key
    - Message (recipient, amount)
    - Distributor's Signature |

When the transaction executes, the `Ed25519 Instruction` is processed by the `Ed25519Program`. If the signature is valid, the instruction execution succeeds. However, if the signature is invalid, it aborts the transaction and logs an error code, which means subsequent instructions (like the `AirdropClaim Instruction`) are not executed.

We’ll demonstrate how this verification works practically later in this article.

## Instruction 2: `AirdropClaim Instruction`

`AirdropClaim Instruction` is a standard Solana transaction instruction sent to the airdrop program to claim the airdrop token. The instruction contains the airdrop program ID, the recipient account, and the instructions sysvar account for introspection.

|  | **AirdropClaim Instruction** |
| --- | --- |
| Program ID |         airdrop program ID  |
| Accounts | [recipient, instructions sysvar account] |
| Instruction Data | No custom data |

The airdrop program will first introspect the ****`Ed25519 Verification Instruction: Instruction 1` using the instruction sysvar to validate that:

- `Ed25519 Verification Instruction: Instruction 1` program ID matches `Ed25519Program`  (`Ed25519SigVerify111111111111111111111111111`).
- The `Ed25519 Verification Instruction: Instruction 1` has no accounts, as expected for the stateless `Ed25519Program`.
- The instruction’s data contains the correct distributor’s public key, signature, and message, matching the expected values.

If the introspection shows that `Ed25519 Verification Instruction: Instruction 1` is valid, the user can claim their airdrop token.

## **Execution flow of the** `Ed25519 Verification Instruction`  **and** `AirdropClaim Instruction`

The diagram below shows a high-level execution flow of the `Ed25519 Verification Instruction` and the `AirdropClaim Instruction` in our program before an airdrop can be claimed.  

The user sends a transaction with two instructions: `Ed25519 Verification Instruction` and `AirdropClaim Instruction`. 

1. The`Ed25519 Verification Instruction` goes to the `Ed25519Program` to verify the distributor’s signature.
2. If the signature verification fails, the entire transaction fails. If it succeeds, the execution flow continues. 
3. The `AirdropClaim Instruction` is then sent to the **Airdrop program**.
4. The **Airdrop program** introspects `Ed25519 Verification Instruction`, checking its program ID, accounts, and data to confirm it was a valid `Ed25519` verification.
5. If introspection confirms `Ed25519 Verification Instruction`, the user can claim their airdrop token.

![A diagram illustrating the execution flow of the Ed25519 Verification Instruction and AirdropClaim Instruction.](https://r2media.rareskills.io/SolanaSignatureVerification/image7.png)

## Signature verification program for airdrop distribution

Let’s write actual code that demonstrates how to use instruction introspection to verify Ed25519 signatures following our airdrop distribution flow. This application has two phases:

1. The **client side** builds the transaction adding `Ed25519 Verification Instruction: Instruction 1` and the `AirdropClaim Instruction: Instruction 2`, then sends the transaction to the network.
2. The **program logic** validates the `Ed25519 Verification Instruction: Instruction 1`through introspection and allows the user to claim their airdrop token.

We’ll implement the client side logic in the test suite, so let’s start by creating the program logic first.

### The program logic: the claim verification

To follow along with this section, ensure you have Solana development environment setup on your machine. Otherwise, read [the first article in the series](https://rareskills.io/post/hello-world-solana) to set it up.

Initialize an Anchor application by running anchor the command:

```bash
 anchor init airdrop-distribution 
```

Update the imports in the `programs/airdrop-distribution/lib.rs` file with these Anchor imports. We need:

- the `ed25519_program` import for our verification,
- the public key for different instances where we need it,
- and then we’ll use the `sysvar` imports for introspection.

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    ed25519_program,
    pubkey::Pubkey, 
    sysvar::instructions as ix_sysvar, 
    sysvar::SysvarId
};
```

Retain your generated `declare_id` 

```rust
declare_id!("Gh2JoycvxfreSgjzhCHuRDK7sZDAbxeo7Pd8GKCoSLmS");
```

Next, we’ll include the rest of the program logic and walk through it step by step. 

The program contains a `claim` function where all the logic lives. Here’s a breakdown of what happens in the function:

1. It loads the instructions `sysvar` to read the full transaction instructions.
2. Finds the index of the current instruction and loads the one immediately before it.
3. Requires that the preceding instruction was sent to the native `Ed25519` program and had no accounts.
4. Parses the `Ed25519 Verification Instruction: Instruction 1` data, then checks the header, validates the number of signatures, and extracts the offsets.
5. Verifies that all offsets in the header point to data within the same instruction and point specifically to the signature, public key, and message.
6. Reconstructs the distributor’s public key from the data and checks it matches the expected distributor account.
7. Reconstructs the signed message `[recipient pubkey (32)][amount (u64 little-endian)]` and checks that the recipient from the signed message matches the recipient account in the `AirdropClaim Instruction: Instruction 2`.

```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::{
    ed25519_program,
    pubkey::Pubkey, 
    sysvar::instructions as ix_sysvar, 
    sysvar::SysvarId
};

declare_id!("Gh2JoycvxfreSgjzhCHuRDK7sZDAbxeo7Pd8GKCoSLmS");

#[program]
pub mod airdrop {
    use super::*;

    pub fn claim(ctx: Context<Claim>) -> Result<()> {

        // --- constants for parsing Ed25519 instruction data ---
        const HEADER_LEN: usize = 16;  // fixed-size instruction header
        const PUBKEY_LEN: usize = 32;  // size of an Ed25519 public key
        const SIG_LEN: usize = 64;     // size of an Ed25519 signature
        const MSG_LEN: usize = 40;     // expected message length: [recipient(32) + amount(8)]

        // Load the instruction sysvar account (holds all tx instructions)
        let ix_sysvar_account = ctx.accounts.instruction_sysvar.to_account_info();

        // Index of the current instruction in the transaction
        let current_ix_index = ix_sysvar::load_current_index_checked(&ix_sysvar_account)
            .map_err(|_| error!(AirdropError::InvalidInstructionSysvar))?;

        // The Ed25519 verification must have run just before this instruction
        require!(current_ix_index > 0, AirdropError::InvalidInstructionSysvar);

        // Load the immediately preceding instruction (the Ed25519 ix)
        let ed_ix = ix_sysvar::load_instruction_at_checked(
            (current_ix_index - 1) as usize,
            &ix_sysvar_account,
        )
        .map_err(|_| error!(AirdropError::InvalidInstructionSysvar))?;

        // Ensure it is the Ed25519 program and uses no accounts (stateless check)
        require!(ed_ix.program_id == ed25519_program::id(), AirdropError::BadEd25519Program);
        require!(ed_ix.accounts.is_empty(), AirdropError::BadEd25519Accounts);
       
        // Ed25519 Verification Instruction data
        let data = &ed_ix.data;

        // --- parse Ed25519 instruction format ---
        // First byte: number of signatures (must be 1)
        // Rest of header: offsets describing where signature, pubkey, and message are
        require!(data.len() >= HEADER_LEN, AirdropError::InvalidInstructionSysvar);
        let sig_count = data[0] as usize;
        require!(sig_count == 1, AirdropError::InvalidInstructionSysvar);

        // helper to read u16 offsets from the header (little-endian)
        let read_u16 = |i: usize| -> Result<u16> {
            let start = 2 + 2 * i;
            let end = start + 2;
            let src = data
                .get(start..end)
                .ok_or(error!(AirdropError::InvalidInstructionSysvar))?;
            let mut arr = [0u8; 2];
            arr.copy_from_slice(src);
            Ok(u16::from_le_bytes(arr))
        };

        // Extract the offsets for signature, pubkey, and message
        let signature_offset = read_u16(0)? as usize;
        let signature_ix_idx = read_u16(1)? as usize;
        let public_key_offset = read_u16(2)? as usize;
        let public_key_ix_idx = read_u16(3)? as usize;
        let message_offset = read_u16(4)? as usize;
        let message_size = read_u16(5)? as usize;
        let message_ix_idx = read_u16(6)? as usize;

        // Enforce that all offsets point to the current instruction's data.
        // The Ed25519 program uses u16::MAX as a sentinel value for "current instruction".
        // This prevents the program from accidentally reading signature, public key,
        // or message bytes from some other instruction in the transaction.
        let this_ix = u16::MAX as usize;
        require!(
            signature_ix_idx == this_ix
                && public_key_ix_idx == this_ix
                && message_ix_idx == this_ix,
            AirdropError::InvalidInstructionSysvar
        );

        // Ensure all offsets point beyond the 16-byte header,
        // i.e. into the region containing the signature, public key, and message
        require!(
            signature_offset >= HEADER_LEN 
                 && public_key_offset >= HEADER_LEN 
                 && message_offset >= HEADER_LEN,
            AirdropError::InvalidInstructionSysvar
        );

        // Bounds checks for signature, pubkey, and message slices
        require!(data.len() >= signature_offset + SIG_LEN, AirdropError::InvalidInstructionSysvar);
        require!(data.len() >= public_key_offset + PUBKEY_LEN, AirdropError::InvalidInstructionSysvar);
        require!(data.len() >= message_offset + message_size, AirdropError::InvalidInstructionSysvar);
        require!(message_size == MSG_LEN, AirdropError::InvalidInstructionSysvar);

        // --- reconstruct and validate the distributor's pubkey ---
        let pk_slice = &data[public_key_offset..public_key_offset + PUBKEY_LEN];
        let mut pk_arr = [0u8; 32];
        pk_arr.copy_from_slice(pk_slice);
        let distributor_pubkey = Pubkey::new_from_array(pk_arr);

        if distributor_pubkey != ctx.accounts.expected_distributor.key() {
            return err!(AirdropError::DistributorMismatch);
        }

        // --- reconstruct and validate the signed message ---
        // Format: [recipient pubkey (32 bytes)][amount (u64 little-endian)]
        let msg = &data[message_offset..message_offset + message_size];

        let mut rec_arr = [0u8; 32];
        rec_arr.copy_from_slice(&msg[0..32]);
        let recipient_from_msg = Pubkey::new_from_array(rec_arr);
        if recipient_from_msg != ctx.accounts.recipient.key() {
            return err!(AirdropError::RecipientMismatch);
        }

        let mut amount_bytes = [0u8; 8];
        amount_bytes.copy_from_slice(&msg[32..40]);
        let amount = u64::from_le_bytes(amount_bytes);

        // User can now claim the airdrop token. 
        // The airdrop transfer can now be implemented here.

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Claim<'info> {
    /// The recipient of the airdrop (must match the recipient in the signed message)
    #[account(mut)]
    pub recipient: Signer<'info>,

    /// Expected distributor pubkey (checked against signed message, not Anchor)
    /// CHECK: Validated manually against the parsed message
    pub expected_distributor: UncheckedAccount<'info>,

    /// The sysvar containing the full transaction's instructions
    /// CHECK: Validated by requiring its well-known address
    #[account(address = ix_sysvar::Instructions::id())]
    pub instruction_sysvar: AccountInfo<'info>,

    /// System program used for the transfer
    pub system_program: Program<'info, System>,
}

#[error_code]
pub enum AirdropError {
    #[msg("Invalid instruction sysvar")]
    InvalidInstructionSysvar,
    #[msg("Expected Ed25519 program id")]
    BadEd25519Program,
    #[msg("Bad Ed25519 accounts")]
    BadEd25519Accounts,
    #[msg("Distributor public key mismatch")]
    DistributorMismatch,
    #[msg("Recipient mismatch in message")]
    RecipientMismatch,
}
```

Let’s explain the key parts of the above code. We’ll cover:

1. How the above code loads the `Ed25519 Verification Instruction: Instruction 1` from the sysvar account using relative instruction indexing helper functions provided by the Solana Rust SDK
2. Accessing and verifying the `Ed25519 Verification Instruction: Instruction 1` data
3. Retrieving the signature count and offsets in the header region
4. Validation to ensure we are accessing accurate signature, public key, and message in the current transaction
5. Accessing the distributor’s signature, public key, and the message in the instruction data

We’ll share screenshots of each key part of the program code above and discuss it in the following sections.

### 1. Introspection: Loading and validating the `Ed25519 Verification Instruction: Instruction 1`

The screenshot below from our program code shows how we use instruction introspection through the instruction sysvar to verify the `Ed25519 Verification Instruction: Instruction 1`. 

1. We call `load_current_index_checked()` to get the index of the current instruction and `load_instruction_at_checked()` to load the immediately preceding instruction.
2. Once we have the preceding instruction (`Ed25519 Verification Instruction: Instruction 1`), we: 
    - verify that its program ID matches the `Ed25519Program`. This ensures the instruction is indeed an Ed25519 signature verification.
    - and confirm that the instruction account list is empty.
3. Once these checks succeed, we extract the instruction’s data, which is a vector and bind it to the variable `data`.

![A screenshot showing a code snippet for how to load and validate the Ed25519 Verification Instruction](https://r2media.rareskills.io/SolanaSignatureVerification/image1.png)

Now, we’ve succeeded in verifying the top level `ed2559Program` instruction information: the ID and the accounts. We’ve also grabbed the `Ed25519 Verification Instruction: Instruction 1` data, so, the next step is to verify the content of the data. The data is a vector of `u8` data type. 

### 2. Accessing and verifying the `Ed25519 Verification Instruction: Instruction 1` data

We expect the instruction data to encode, in order: a header that specifies the signature count and offsets for the following fields; the distributor’s public key; the message; and the distributor’s Ed25519 signature.

![A screenshoot of a table showing the content of an Ed25519 instruction](https://r2media.rareskills.io/SolanaSignatureVerification/image6.png)

Now, we’ll step through the next part of our code to see how the airdrop program is accessing and verifying the `Ed25519 Verification Instruction: Instruction 1` data.

### **3. Retrieving the signature count and offsets in the header region**

The code in the screenshot below extracts the signature count, the offsets and the indexes pointing to where each element is in the `Ed25519 Verification Instruction: Instruction 1` data vector.

In the header, the signature count should be in the first index, we get that with `data[0]`. The expectation is that the count is 1 because there should be only one distributor signature. We enforce that with a `require` statement.

After that, the header contains offset and index values which tell us where to find the distributor’s public key, the signature, and the message within the instruction data.

To parse them, we define a closure `read_u16` that steps through the data buffer two bytes at a time, returning each offset as a `u16`. This makes it easier to reconstruct a consistent instruction data layout.

![A screenshot showing the snippet for retrieving the signature count and offsets in the header region of an instruction data.](https://r2media.rareskills.io/SolanaSignatureVerification/image2.png)

### **4.** Validation to ensure we are accessing accurate signature, public key, and message in the current instruction

At this point, we have the signature count and the offsets but we need to make sure:

1. We are interacting with the instruction we loaded from the sysvar as the current instruction. Recall that the index of the signature (`signature_ix_idx`), public key (`public_key_ix_idx`), and message (`message_ix_idx`) in the current instruction data are set to `u16::MAX` in the Ed25519 source code. Any other value would point to another instruction in the transaction.
2. The offsets are pointing beyond the 16 byte header and into the part of the vector that contains the signature, public key, and the message.

![A screenshot explaining: Validation to ensure we are accessing accurate signature, public key, and message in the current instruction following our program code.](https://r2media.rareskills.io/SolanaSignatureVerification/image3.png)

### **5. Accessing the distributor’s signature, public key, and the message in the instruction data vector**

The screenshot below shows how we use the offsets parsed from the `Ed25519 Verification Instruction: Instruction 1` data header to locate the distributor’s public key and message content (recipient and amount) within instruction data and validating them against the version provided by the user in the`AirdropClaim Instruction: Instruction 2`.

- The first marked region shows how we slice out the distributor’s public key from the `Ed25519 Instruction` data, reconstruct it as a 32-byte `Pubkey`, and compare it to the `expected_distributor` public key from the distributor account in the `AirdropClaim Instruction: Instruction 2`.
- The second marked region shows how we slice out the signed message (recipient + amount), reconstruct the recipient pubkey, and verify it matches the `recipient` account in the `AirdropClaim Instruction: Instruction 2`.

If both checks succeed, the signature verification is complete. At this point you could implement the token transfer to the recipient. Since this article focuses on verification, we have not implemented the transfer.

![A screenshot showing showing how to access the distributor’s signature, public key, and the message in the instruction data vector](https://r2media.rareskills.io/SolanaSignatureVerification/image4.png)

## The **client side: constructing the transaction off-chain**

We’ve seen how the signature verification works. Now, let’s test it by creating a transaction that will contain the two instructions — `Ed25519 Verification Instruction: Instruction 1` and the `AirdropClaim Instruction: Instruction 2`. 

**Dependencies**

We’ll use the `tweetnacl` cryptographic library to create the distributor signature, so install it by running the command below:

```bash
yarn add tweetnacl
```

Once that is done, add the `tweetnacl` to your imports in `tests/airdrop-distribution.ts` alongside the following imports as shown below. We’ll use the `Ed25519Program` dependency to create the first instruction for the verification, while `TransactionInstruction` is the expected standard transaction instruction type.

```rust
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { expect } from "chai";

// Add the following 
import { Airdrop } from "../target/types/airdrop"; // The IDL
import { 
    PublicKey, 
    Keypair, 
    SystemProgram, 
    Transaction, 
    **TransactionInstruction, 
    Ed25519Program** 
} from "@solana/web3.js";
import * as nacl from "tweetnacl";
```

We’ll have four test case scenarios:

1. **Valid claim**: distributor signs the correct recipient and amount, `Ed25519Program` instruction runs before the `claim` instruction then the transaction succeeds.
2. **Wrong order**: `claim` instruction comes before `Ed25519Program`, the transaction fails with `InvalidInstructionSysvar`.
3. **Wrong distributor**: signature doesn’t match `expectedDistributor` signature, the transaction fails with `DistributorMismatch`.
4. **Wrong recipient**: signed recipient differs from the user trying to claim the airdrop’s signature, the transaction fails with `RecipientMismatch`.
5. **Multiple claims:** a test case to show that an attempt to cheat the system by constructing multiple `AirdropClaim Instruction` will fail. That’s because the program’s introspection logic only looks at the immediately preceding `Ed25519 Verification Instruction: Instruction 1`, so the second `AirdropClaim Instruction` will fail. 

Start by setting up the test first to use the local cluster and set up test accounts for the distributor, the recipient and an invalid distributor account for negative test cases.

```tsx
// ...
describe("airdrop", () => {
  // Configure the client to use the local cluster
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.Airdrop as Program<Airdrop>;
  const provider = anchor.getProvider();

  // Test accounts
  let distributorKeypair: Keypair;
  let recipientKeypair: Keypair;
  let invalidDistributorKeypair: Keypair;

  before(async () => {
    // Generate test keypairs
    distributorKeypair = Keypair.generate();
    recipientKeypair = Keypair.generate();
    invalidDistributorKeypair = Keypair.generate();
  });

```

Next, we’ll add a helper function that builds the `Ed25519 Verification Instruction: Instruction 1`. It constructs the message from the recipient and amount, signs it with the distributor’s key, and then uses `Ed25519Program.createInstructionWithPublicKey` to return a `TransactionInstruction` the runtime can verify.

```tsx
function createEd25519Instruction(
  distributorKeypair: Keypair,
  recipientPubkey: PublicKey,
  amount: number
): TransactionInstruction {
  // Build the message: 32 bytes recipient pubkey + 8 bytes amount
  const message = Buffer.alloc(40);
  recipientPubkey.toBuffer().copy(message, 0);
  message.writeBigUInt64LE(BigInt(amount), 32);

  // Sign the message with distributor
  const signature = nacl.sign.detached(message, distributorKeypair.secretKey);

  // Use the helper to build the instruction
  return Ed25519Program.createInstructionWithPublicKey({
    publicKey: distributorKeypair.publicKey.toBytes(),
    message,
    signature,
  });
}
```

We’ll re-use the above function in our test cases to create the `Ed25519 Verification Instruction: Instruction 1`. Let’s start with our first test case, which is a valid airdrop claim that should succeed.

We create two instructions: `Ed25519 Verification Instruction: Instruction 1` and the `AirdropClaim Instruction: Instruction 2`. We pass the distributor, recipient, and instruction sysvar accounts to the program’s `claim` function, as defined earlier. Then we send the transaction and confirm it succeeded. On success, it returns a transaction ID; otherwise, we get an error.

```tsx
  it("Successfully claims airdrop with valid signature", async () => {
  const claimAmount = 1000000;
    
  // Create Ed25519 Signature Verification Instruction: Instruction 1 
  const ed25519Ix = createEd25519Instruction(
    distributorKeypair,
    recipientKeypair.publicKey,
    claimAmount
  );
 
  // Create the AirdropClaim Instruction: Instruction 2
  const claimIx = await program.methods
    .claim()
    .accountsPartial({
      recipient: recipientKeypair.publicKey,
      expectedDistributor: distributorKeypair.publicKey,
      instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
    })
    .instruction();

  const tx = new Transaction();
  tx.add(ed25519Ix); // Add Instruction 1 to the transaction 
  tx.add(claimIx); // Add Instruction 2 to the transaction
 
  // Just expect the transaction to succeed
  expect(await provider.sendAndConfirm(tx, [recipientKeypair])).to.not.be.empty;
});
```

The failure cases will involve the same process, we’ll only need to add invalid data that will cause them to fail. So, here’s the complete test code with explanatory comments. 

```tsx
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Airdrop } from "../target/types/airdrop";
import { PublicKey, Keypair, SystemProgram, Transaction, TransactionInstruction, Ed25519Program } from "@solana/web3.js";
import { expect } from "chai";
import * as nacl from "tweetnacl";

describe("airdrop", () => {
  // Configure the client to use the local cluster
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.Airdrop as Program<Airdrop>;
  const provider = anchor.getProvider();

  // Test accounts
  let distributorKeypair: Keypair;
  let recipientKeypair: Keypair;
  let invalidDistributorKeypair: Keypair;

  before(async () => {
    // Generate test keypairs
    distributorKeypair = Keypair.generate();
    recipientKeypair = Keypair.generate();
    invalidDistributorKeypair = Keypair.generate();
  });

function createEd25519Instruction(
  distributorKeypair: Keypair,
  recipientPubkey: PublicKey,
  amount: number
): TransactionInstruction {
  // Build the message: 32 bytes recipient pubkey + 8 bytes amount
  const message = Buffer.alloc(40);
  recipientPubkey.toBuffer().copy(message, 0);
  message.writeBigUInt64LE(BigInt(amount), 32);

  // Sign the message with distributor
  const signature = nacl.sign.detached(message, distributorKeypair.secretKey);

  // Use the helper to build the instruction
  return Ed25519Program.createInstructionWithPublicKey({
    publicKey: distributorKeypair.publicKey.toBytes(),
    message,
    signature,
  });
}

  it("Successfully claims airdrop with valid signature", async () => {
	  const claimAmount = 1000000;
	
	  const ed25519Ix = createEd25519Instruction(
	    distributorKeypair,
	    recipientKeypair.publicKey,
	    claimAmount
	  );
	
	  const claimIx = await program.methods
	    .claim()
	    .accountsPartial({
	      recipient: recipientKeypair.publicKey,
	      expectedDistributor: distributorKeypair.publicKey,
	      instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
	    })
	    .instruction();
	
	  const tx = new Transaction();
	  tx.add(ed25519Ix);
	  tx.add(claimIx); // AirdropClaim Instruction: Instruction 2
	  
	  // Just expect the transaction to succeed
	  expect(await provider.sendAndConfirm(tx, [recipientKeypair])).to.not.be.empty;
	});

  it("Fails when Ed25519 instruction is not first", async () => {
    const claimAmount = 1000000;

    const claimIx = await program.methods
      .claim()
      .accountsPartial({
        recipient: recipientKeypair.publicKey,
        expectedDistributor: distributorKeypair.publicKey,
        instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();

    const ed25519Ix = createEd25519Instruction(
      distributorKeypair,
      recipientKeypair.publicKey,
      claimAmount
    );

    // Create transaction with claim first, then Ed25519 (wrong order)
    const tx = new Transaction();
    tx.add(claimIx);
    tx.add(ed25519Ix);

    try {
      await provider.sendAndConfirm(tx, [recipientKeypair]);
      expect.fail("Should have failed with wrong instruction order");
    } catch (error) {
      expect(error.message).to.include("InvalidInstructionSysvar");
    }
  });

  it("Fails with distributor mismatch", async () => {
    const claimAmount = 1000000;

    // Create Ed25519 instruction with wrong distributor
    const ed25519Ix = createEd25519Instruction(
      invalidDistributorKeypair, // Wrong distributor signs
      recipientKeypair.publicKey,
      claimAmount
    );

    const claimIx = await program.methods
      .claim()
      .accountsPartial({
        recipient: recipientKeypair.publicKey,
        expectedDistributor: distributorKeypair.publicKey, // But we expect the correct one
        instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();

    const tx = new Transaction();
    tx.add(ed25519Ix);
    tx.add(claimIx);

    try {
      await provider.sendAndConfirm(tx, [recipientKeypair]);
      expect.fail("Should have failed with distributor mismatch");
    } catch (error) {
      expect(error.message).to.include("DistributorMismatch");
    }
  });

  it("Fails with recipient mismatch", async () => {
    const claimAmount = 1000000;
    const wrongRecipient = Keypair.generate();

    // Create Ed25519 instruction with wrong recipient in message
    const ed25519Ix = createEd25519Instruction(
      distributorKeypair,
      wrongRecipient.publicKey, // Wrong recipient in signed message
      claimAmount
    );

    const claimIx = await program.methods
      .claim()
      .accountsPartial({
        recipient: recipientKeypair.publicKey,
        expectedDistributor: distributorKeypair.publicKey,
        instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
      })
      .instruction();

    const tx = new Transaction();
    tx.add(ed25519Ix);
    tx.add(claimIx);

    try {
      await provider.sendAndConfirm(tx, [recipientKeypair]);
      expect.fail("Should have failed with recipient mismatch");
    } catch (error) {
      expect(error.message).to.include("RecipientMismatch");
    }
  });

   it("Fails when multiple claim instructions try to reuse the same Ed25519 signature", async () => {
	    const claimAmount = 1000000;
	
	    // Create a single Ed25519 instruction
	    const ed25519Ix = createEd25519Instruction(
	      distributorKeypair,
	      recipientKeypair.publicKey,
	      claimAmount
	    );
	
	    // First claim instruction (valid)
	    const claimIx1 = await program.methods
	      .claim()
	      .accountsPartial({
	        recipient: recipientKeypair.publicKey,
	        expectedDistributor: distributorKeypair.publicKey,
	        instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
	      })
	      .instruction();
	
	    // Second claim instruction (tries to reuse the same Ed25519)
	    const claimIx2 = await program.methods
	      .claim()
	      .accountsPartial({
	        recipient: recipientKeypair.publicKey,
	        expectedDistributor: distributorKeypair.publicKey,
	        instructionSysvar: anchor.web3.SYSVAR_INSTRUCTIONS_PUBKEY,
	      })
	      .instruction();
	
	    const tx = new Transaction();
	    tx.add(ed25519Ix);
	    tx.add(claimIx1);
	    tx.add(claimIx2);
	
	    try {
	      await provider.sendAndConfirm(tx, [recipientKeypair]);
	      expect.fail("Should have failed because multiple claims tried to reuse the same signature");
	    } catch (error) {
	      // The second claim fails because its immediately preceding instruction
	      // is not the Ed25519 verification, so the program throws
	      expect(error.message).to.include("BadEd25519Program");
	    }
  });

});
```

Let’s run the test with the command below:

```rust
anchor test
```

And the result should look like this:

![A screenshot showing the result of a successful test](https://r2media.rareskills.io/SolanaSignatureVerification/image5.png)

Our implementation so far has been focused on signature verification. Understand that this example is for learning purposes, you should consider standard program security best practices when creating and sending real transactions.

There have been cases where **wrong offset implementation** introduced vulnerabilities. One such example is covered in the article *“[Wrong Offset: Bypassing Signature Verification.](https://blog.asymmetric.re/wrong-offset-bypassing-signature-verification-in-relay/)”* While what we’ve learned in this article is not affected by that vulnerability, it’s worth being aware of the potential risk.

*This article is part of a [tutorial series on Solana](https://rareskills.io/solana-tutorial).*
