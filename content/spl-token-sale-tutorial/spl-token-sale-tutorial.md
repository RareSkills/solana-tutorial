# Token Sale with Total Supply Tutorial

A token sale program is a smart contract that sells a specific token, usually in exchange for a native token like SOL, at a fixed price. The sale runs until a predefined supply is sold or the owner acts to end the sale.

Our implementation follows this flow: 

1. A user deposits SOL based on our rate, e.g, 1 SOL to 100 tokens.
2. The program stores SOL in a treasury [Program Derived Address (PDA)](https://rareskills.io/post/solana-pda), a program-controlled account.
3. Once the SOL is received, the tokens are minted to the user.
4. The sale continues until reaching a predefined supply cap.
5. An admin can withdraw the collected SOL from the treasury.

## Creating the Token Sale program

**The Solana program we’ll build mints SPL tokens directly to buyers, without requiring us to sign each transaction as the mint authority. This is the standard approach—otherwise the admin would need to manually approve every purchase, which isn’t practical.**

### Creating the accounts required for the token sale

First, create a new `token_sale` program with Anchor and replace the boilerplate code in `programs/token_sale/src/lib.rs` with the code below.
The code below imports our program dependencies and defines an `initialize` function. The function does the following:

1. Sets up the admin account to control the treasury withdrawals
2. Creates a mint account for the new token we are selling
3. Creates a treasury account to collect SOL from token purchases

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};
use anchor_spl::token::{mint_to, Mint, MintTo, Token, TokenAccount};

declare_id!("Gm8bFHtX3TapZDqA2tjviP1Qn1f8bLjTf8tbhFcgzcFs"); // REPLACE THIS WITH YOUR PROGRAM ID OR RUN `anchor sync`

// Tokens per SOL, i.e., 1 SOL == 100 of our tokens
const TOKENS_PER_SOL: u64 = 100;
// Max supply: 1000 tokens (with 9 decimals)
const SUPPLY_CAP: u64 = 1000e9 as u64;

#[program]
pub mod token_sale {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        // Set the admin key
       ctx.accounts.admin_config.admin = ctx.accounts.admin.key();
        Ok(())
    }
}

```

In the code above, we define constants for our token sale program: `TOKENS_PER_SOL = 100` and `SUPPLY_CAP = 1000` (with 9 decimals).

Next, add the `Initialize` account struct for our function. It contains the following accounts:

- `admin`: The account that pays for transaction fees and serves as the program administrator
- `admin_config`: A PDA that stores the admin's public key, so later during withdrawals we can verify the signer is the same admin (like checking `msg.sender == admin` in Solidity, where `admin` is a state variable that stores an admin's public key). We use a PDA with seed `"admin_config"` so the address can be deterministically derived without needing off-chain storage.
- `mint`: A self-referential mint PDA that serves as both the token mint and its own authority (we will explain the concept later)
- `treasury`: A PDA that holds SOL collected from token sales
- Finally, we pass the Token Program and System Program that we interact with.

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub admin: Signer<'info>, // The transaction signer

    #[account(
        init,
        payer = admin,
        space = 8+AdminConfig::INIT_SPACE, // 8 is for the discriminator
        seeds = [b"admin_config"],
        bump,
    )]
    pub admin_config: Account<'info, AdminConfig>,

    #[account(
        init,
        payer = admin,
        seeds = [b"token_mint"],
        bump,
        mint::decimals = 9,
        mint::authority = mint.key(),
    )]
    pub mint: Account<'info, Mint>,

    /// CHECK: PDA for treasury
    #[account(
        seeds = [b"treasury"],
        bump
    )]
    pub treasury: AccountInfo<'info>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

// Stores the admin public key
#[account]
#[derive(InitSpace)] // This is a derive attribute macro provided by anchor, it calculates the space needed for the account and gives us access to AdminConfig::INIT_SPACE, as used above
pub struct AdminConfig {
    pub admin: Pubkey,
}

#[error_code]
pub enum Errors {
    #[msg("Max token supply limit reached")]
    SupplyLimit,

    #[msg("Math overflow")]
    Overflow,

    #[msg("Only admin can withdraw")]
    UnauthorizedAccess,

    #[msg("Not enough SOL in treasury")]
    InsufficientFunds,
}

```

### Understanding the Initialize struct accounts

Let's break down each account in the `Initialize` account struct and understand their purpose:

**admin config**

`admin_config`: This account holds the admin's public key (defined by the `AdminConfig` struct) and is used to make sure only the admin can withdraw SOL from the treasury. We create it as a PDA using the seed `"admin_config"` so it can be easily derived later without needing to store its address off-chain.

By using a PDA for the admin config, we ensure the address is deterministic and can always be re-derived using `findProgramAddressSync`.
We don't need to store the address or generate and manage a keypair during initialization

![A screenshot showing constraints for the admin_config account initialization](https://r2media.rareskills.io/SolanaSPLTokenSale/image8.png)

![Account definition of AdminConfig](https://r2media.rareskills.io/SolanaSPLTokenSale/image1.png)

**mint account**

This is the mint account for our SPL token (the token being sold). We create it as a PDA so the program can sign for it later (we will explain this later in this article).

The account doesn’t exist on‑chain until we invoke `initialize`. In that call, Anchor will:

1. Compute the mint PDA address with the seed `"token_mint"` and the program ID
2. Create the account with `mint::decimals = 9` (this is what we set, as shown below)
3. Set the mint’s authority to itself (`mint::authority = mint.key()`). This part is important because by making the PDA its own authority, only our program, using the same seed and bump, can sign `mint_to` instructions (again, we will explain how this works later in this article).

**treasury**

This PDA is used solely to hold the SOL (lamports) sent by users during the sale.

![Initialization parameters for the PDA that holds SOL](https://r2media.rareskills.io/SolanaSPLTokenSale/image2.png)

Now update the `programs/token_sale/Cargo.toml` file with the following.

```toml
[package]
name = "token_sale"
version = "0.1.0"
description = "Created with Anchor"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "token_sale"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"] # "anchor-spl/idl-build" was added

[dependencies]
anchor-lang = "0.31.0"
anchor-spl = "0.31.0" # THIS WAS ADDED

```

Now update the test for our program.

This test is very similar to what we saw in previous tutorials. It simply calls our program’s `initialize` instruction with the required accounts and asserts the properties of the newly created mint account (token).

```tsx

import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import {
  createAssociatedTokenAccount,
  getAccount,
  getMint,
  TOKEN_PROGRAM_ID
} from "@solana/spl-token";
import * as web3 from "@solana/web3.js";
import { assert } from "chai";
import { TokenSale } from "../target/types/token_sale";

describe("token_sale", async () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.TokenSale as Program<TokenSale>;
  const connection = provider.connection;

  const adminKp = provider.wallet.payer;
  const buyer = adminKp; // Using the same keypair as both admin and buyer for testing
  const TOKENS_PER_SOL = 100;

  let mint: anchor.web3.PublicKey;
  let treasuryPda: anchor.web3.PublicKey;
  let buyerAta: anchor.web3.PublicKey;
  let adminConfigPda: anchor.web3.PublicKey;

    it("creates mint", async () => {
    [mint] = web3.PublicKey.findProgramAddressSync(
      [Buffer.from("token_mint")],
      program.programId
    );

    [treasuryPda] = web3.PublicKey.findProgramAddressSync(
      [Buffer.from("treasury")],
      program.programId
    );

    [adminConfigPda] = web3.PublicKey.findProgramAddressSync(
      [Buffer.from("admin_config")],
      program.programId
    );

    const tx = await program.methods
      .initialize()
      .accounts({
        admin: adminKp.publicKey,
        adminConfig: adminConfigPda,
        mint: mint,
        treasury: treasuryPda,
        tokenProgram: TOKEN_PROGRAM_ID,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([adminKp])
      .rpc();

    console.log("initialize tx:", tx);

    const mintInfo = await getMint(connection, mint);
    assert.equal(mintInfo.mintAuthority.toBase58(), mint.toBase58());
    assert.equal(Number(mintInfo.supply), 0);
    assert.equal(mintInfo.decimals, 9);
  });
});

```

Run `npm install @solana/spl-token` to update dependency.

Run the test, and it passes.

![image.png](https://r2media.rareskills.io/SolanaSPLTokenSale/image3.png)

### Buying 100 tokens with 1 SOL

We have set up our Token Sale program. We will now add a function to mint new token units for sale, so users can purchase our token.

The code does the following:

1. Calculate the number of tokens to mint based on the lamport input.
2. Check that we don’t exceed total supply.
3. Transfer SOL from the buyer to the treasury.
4. Prepare signer seeds so the program can sign on behalf of the mint PDA (more on this after the code block below).
5. Set up the mint instruction with the mint account as its own authority.
6. Create a CPI context using the signer seeds.
7. Mint the tokens to the buyer’s token account.

```rust

    pub fn mint(ctx: Context<MintTokens>, lamports: u64) -> Result<()> {
        // Calculate how many tokens to mint (lamports * TOKENS_PER_SOL)
        let amount = lamports
            .checked_mul(TOKENS_PER_SOL)
            .ok_or(Errors::Overflow)?; // If overflow, return error

        // Ensure we don't exceed the max supply
        let current_supply = ctx.accounts.mint.supply;
        let new_supply = current_supply.checked_add(amount).ok_or(Errors::Overflow)?; // If overflow, return error
        require!(new_supply <= SUPPLY_CAP, Errors::SupplyLimit);

        // Send SOL to treasury
        let transfer_instruction = Transfer {
            from: ctx.accounts.buyer.to_account_info(),
            to: ctx.accounts.treasury.to_account_info(),
        };

        let cpi_context = CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            transfer_instruction,
        );
        transfer(cpi_context, lamports)?;

				// Create signer seeds for the mint PDA
        let bump = ctx.bumps.mint;
        let signer_seeds: &[&[&[u8]]] = &[&[b"token mint".as_ref(), &[bump]]];

        // Setup mint instruction with mint as its own authority
        let mint_to_instruction = MintTo {
            mint: ctx.accounts.mint.to_account_info(),
            to: ctx.accounts.buyer_token_account.to_account_info(),
            authority: ctx.accounts.mint.to_account_info(),
        };

        // Create CPI context with `new_with_signer` - allows our token sale program to sign for the mint PDA. This works because the Solana runtime verifies that our program derived the mint PDA with these seeds and bump
        // See here for more: <https://github.com/solana-foundation/developer-content/blob/main/content/guides/getstarted/how-to-cpi-with-signer.md>
        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            mint_to_instruction,
            signer_seeds,
        );
        mint_to(cpi_ctx, amount)?;

        Ok(())
    }

```

### **Why we set the mint as its own authority**

In the `mint()` function above, you can see how we use `CpiContext::new_with_signer` to mint tokens. This works because of how we set up the mint account earlier. Recall that during initialization, we set `mint::authority = mint.key()`, making the mint PDA its own authority.

Here's why this pattern is essential.

**The challenge of token minting authorities**

Token minting requires authority control. Normally, you'd assign a specific keypair as the mint authority, and that keypair would need to sign every `mint_to` instruction. While this provides security, it creates a practical problem: we'd have to manage this keypair and ensure it's available to sign every mint operation.

However, this approach isn’t suitable for an automated token sale. Users can’t purchase tokens unless the authority keypair is available to sign each minting. This defeats the purpose of creating a permissionless system.

**How PDAs solve the authority problem**

Instead of using a traditional keypair, we make the mint PDA its own authority. This seems confusing at first: how can a PDA sign transactions when PDAs don’t have private keys?

The solution lies in PDA signing. When our program wants to mint tokens, it uses `CpiContext::new_with_signer` with the exact seeds used to create the mint PDA (`"token_mint"` + bump). The Solana runtime recognizes that our program derived this PDA with these specific seeds, so it allows our program to act as the signer for that PDA.

This creates a useful pattern:

- The mint authority is the PDA address (not a keypair)
- Only our program can "sign" for this PDA using the correct seeds
- Our program indirectly acts as the mint authority (because it owns the mint PDA) and can mint tokens on demand without the need for a dedicated external signer
- No one else can mint these tokens, even if they discover the seeds (the token here can only be minted when a user purchases it via our program’s `mint` function)

**Why not make the admin the mint authority?**

We could have set `mint::authority = admin.key()`, and make the admin the mint authority. But as said earlier, then the admin would need to sign every mint transaction.

Now let’s continue with the program and add the `MintTokens` accounts struct.

`MintTokens` specifies the accounts involved during token sale/mint.

- `buyer`: The account that make the token purchase and also sign the transaction.
- `mint`: Our SPL token for sale.
- `buyer_ata`: The buyer’s associated token account to receive the minted token units.
- `treasury`: The account to receive SOL from the token sale.
- The final two accounts, **Token program** (for minting) and **System program** (for SOL transfers), are the native programs we interact with.

```rust

#[derive(Accounts)]
pub struct MintTokens<'info> {
    #[account(mut)]
    pub buyer: Signer<'info>,

    #[account(
        mut,
        seeds = [b"token mint"],
        bump
    )]
    pub mint: Account<'info, Mint>,

    #[account(
        mut,
        token::mint = mint,
        token::authority = buyer,
    )]
    pub buyer_ata: Account<'info, TokenAccount>,

    /// CHECK: PDA for treasury
    #[account(
        mut,
        seeds = [b"treasury"],
        bump
    )]
    pub treasury: AccountInfo<'info>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

```

Add the following custom error to the program code. The errors here are used in the mint function and the next function we will add.

```rust

#[error_code]
pub enum Errors {
    #[msg("Max token supply limit reached")]
    SupplyLimit,

    #[msg("Math overflow")]
    Overflow,

    #[msg("Only admin can withdraw")]
    UnauthorizedAccess,

    #[msg("Not enough SOL in treasury")]
    InsufficientFunds,
}

```

Now update the test with the code below.

This test does the following:

1. Creates a buyer's ATA for our token
2. Purchases 1 SOL worth of our tokens (100 tokens) by calling our program's mint function
3. Asserts that the treasury has the correct amount of SOL from the trade
4. Asserts that the buyer's ATA has the correct amount of minted tokens after the purchase

```tsx

  it("buys tokens", async () => {
    const solToSend = new anchor.BN(1e9); // 1 SOL
    const expectedTokenAmount = Number(solToSend) * TOKENS_PER_SOL; // 1*100 tokens

    const initialTreasuryBalance = await connection.getBalance(treasuryPda);

    // Create buyer's ata
    buyerAta = await createAssociatedTokenAccount(
      provider.connection,
      buyer,
      mint,
      buyer.publicKey,
      undefined,
      TOKEN_PROGRAM_ID
    );

    const buyerAtaInfo = await getAccount(connection, buyerAta, undefined, TOKEN_PROGRAM_ID);
    const initialBuyerAtaBalance = Number(buyerAtaInfo.amount);

		// Call our program's mint function to purchase tokens
    const tx = await program.methods
      .mint(solToSend)
      .accounts({
        buyer: buyer.publicKey,
        mint: mint,
        buyerAta: buyerAta,
        treasury: treasuryPda,
        tokenProgram: TOKEN_PROGRAM_ID,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    console.log("mint tx:", tx);
    console.log("Sent", lamportsToSol(solToSend), "SOL, expecting", toDisplayAmount(expectedTokenAmount), "tokens");

    const newTreasuryBalance = await connection.getBalance(treasuryPda);
    assert.equal(
      newTreasuryBalance - initialTreasuryBalance,
      Number(solToSend),
      "SOL was not correctly transferred to treasury"
    );

    const updatedBuyerAtaInfo = await getAccount(connection, buyerAta, undefined, TOKEN_PROGRAM_ID);
    const newBuyerAtaBalance = Number(updatedBuyerAtaInfo.amount);

    assert.equal(
      newBuyerAtaBalance - initialBuyerAtaBalance,
      expectedTokenAmount,
      "Tokens were not correctly minted"
    );
  });

```

Now run the test, it passes.

![Screenshot of the terminal showing a successful token mint.](https://r2media.rareskills.io/SolanaSPLTokenSale/image4.png)

### Ensure token sale ends when supply cap is reached

Remember that the supply cap is 1000 tokens and we are minting 100 with 1 SOL in the test above. We will try to mint 920 tokens with 9.2 SOL to confirm that token sale prevents minting more than the supply limit.

Add the following test block.

This test asserts that the token sale fails when we try to buy more than the 1000 limit.

```tsx

  it("stops minting when supply cap is reached", async () => {
    const mintInfo = await getMint(connection, mint, undefined, TOKEN_PROGRAM_ID);
    const currentSupply = Number(mintInfo.supply);

    const SUPPLY_CAP = toRawTokenAmount(1000);
    const remainingSupply = SUPPLY_CAP - currentSupply;

    console.log(`Current supply: ${toDisplayAmount(currentSupply)} tokens, Remaining: ${toDisplayAmount(remainingSupply)} tokens`);

    const tokensToMint = remainingSupply + toRawTokenAmount(20);
    const solToSend = new anchor.BN(Math.ceil(tokensToMint / TOKENS_PER_SOL));

    console.log(`Trying to mint ${toDisplayAmount(tokensToMint)} tokens by sending ${lamportsToSol(solToSend)} SOL`);

    try {
      await program.methods
        .mint(solToSend)
        .accounts({
          buyer: buyer.publicKey,
          mint: mint,
          buyerAta: buyerAta,
          treasury: treasuryPda,
          tokenProgram: TOKEN_PROGRAM_ID,
          systemProgram: anchor.web3.SystemProgram.programId,
        })
        .rpc();

      assert.fail("Minting succeeded but should have failed due to supply cap");
    } catch (error) {
      console.log("Expected error:", error.toString().substring(0, 150) + "...");
      assert.include(
        error.toString(),
        "SupplyLimit",
        "Expected supply limit error not received"
      );
      console.log("Supply cap limit correctly enforced");
    }
  });

```

Run the test and it should pass.

![Screenshot of terminal showing the supply limit works](https://r2media.rareskills.io/SolanaSPLTokenSale/image5.png)

### Withdrawing treasury account SOL balance

So far, we have set up tests to initialize our program and purchase our tokens. Now, we will add code for the token sale program admin (our account) to withdraw collected SOL from the treasury account and test this functionality.

Add the `withdraw_funds` function below to the token sale program. It does the following:

1. Check that the treasury has enough balance for the withdrawal.
2. Prepare signer seeds so the program can sign on behalf of the treasury PDA.
3. Set up a CPI context to call the System Program’s `transfer` instruction.
4. Use `CpiContext::new_with_signer` to allow the program to sign for the treasury.
5. Transfer SOL (lamports) from the treasury to the admin wallet.

**Note:** This uses Solana’s System `transfer`, not the SPL token transfer, because we’re transferring SOL (the native token) rather than an SPL token. SPL transfers require interacting with token accounts, while SOL transfers are handled directly by the system program.

```rust

 pub fn withdraw_funds(ctx: Context<WithdrawFunds>, amount: u64) -> Result<()> {
        // Check balance
        let treasury_balance = ctx.accounts.treasury.lamports();
        require!(treasury_balance >= amount, Errors::InsufficientFunds);

        // Create signer seeds for PDA
        let bump = ctx.bumps.treasury;
        let signer_seeds: &[&[&[u8]]] = &[&[b"treasury".as_ref(), &[bump]]];

        // Prepare the CPI context to System Program::transfer
        // DO NOT CONFUSE THIS WITH SPL TOKEN TRANSFER
        let transfer_instruction = Transfer {
            from: ctx.accounts.treasury.to_account_info(),
            to: ctx.accounts.admin.to_account_info(),
        };

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.system_program.to_account_info(),
            transfer_instruction,
            signer_seeds,
        );
        transfer(cpi_ctx, amount)?; // DO NOT CONFUSE THIS WITH SPL TOKEN TRANSFER

        Ok(())
    }

```

Now add the `WithdrawFunds` account struct, used by the `withdraw_funds` function. It contains the following accounts:

- **`admin`**: The transaction signer, and our admin account.
- **`admin_config`**: The PDA (derived with seed `"admin_config"`) that stores the admin's public key, with a constraint to verify the signer is authorized. We pass it because we need to check that the current signer matches the admin key stored during initialization.
- **`treasury`**: The mutable treasury PDA that holds the SOL to be withdrawn.
- **`system_program`**: The System Program for handling SOL transfers.

```rust

#[derive(Accounts)]
pub struct WithdrawFunds<'info> {
    #[account(mut)]
    pub admin: Signer<'info>,

    #[account(
        constraint = admin_config.admin == admin.key() @ Errors::UnauthorizedAccess // Ensure the signer is authorized
    )]
    pub admin_config: Account<'info, AdminConfig>,

    /// CHECK: PDA for treasury
    #[account(
        mut,
        seeds = [b"treasury"],
        bump
    )]
    pub treasury: AccountInfo<'info>,

    pub system_program: Program<'info, System>,
}

```

Next, update the test.

This test block withdraws half of the treasury balance from the treasury PDA to the admin keypair and asserts that the admin's balance increased by the amount withdrawn, while the treasury's balance decreased by the same amount.

```tsx

  it("allows the admin to withdraw funds from treasury", async () => {
    const initialAdminBalance = await connection.getBalance(adminKp.publicKey);
    const initialTreasuryBalance = await connection.getBalance(treasuryPda);

    console.log("Initial treasury balance:", lamportsToSol(initialTreasuryBalance), "SOL");
    console.log("Initial admin balance:", lamportsToSol(initialAdminBalance), "SOL");

    assert.isAbove(
      initialTreasuryBalance,
      0,
      "Treasury should have funds from previous tests"
    );

    const amountToWithdraw = new anchor.BN(Math.floor(initialTreasuryBalance / 2)); // Withdraw half of the treasury balance

    try {
      const tx = await program.methods
        .withdrawFunds(amountToWithdraw)
        .accounts({
          admin: adminKp.publicKey,
          adminConfig: adminConfigPda,
          treasury: treasuryPda,
          systemProgram: anchor.web3.SystemProgram.programId,
        })
        .rpc();
      console.log("withdrawFunds tx:", tx);

      const newAdminBalance = await connection.getBalance(adminKp.publicKey);
      const newTreasuryBalance = await connection.getBalance(treasuryPda);

      console.log("New treasury balance:", lamportsToSol(newTreasuryBalance), "SOL");
      console.log("New admin balance:", lamportsToSol(newAdminBalance), "SOL");

      // assert that the treasury balance decreased by the amount we withdrew, which is half of the initial treasury balance
      assert.approximately(
        initialTreasuryBalance - newTreasuryBalance,
        Number(amountToWithdraw),
        10000,
        "Treasury balance did not decrease by approximately the correct amount"
      );

      // assert that the admin balance increased by the amount we withdrew
      assert.isTrue(
        newAdminBalance > initialAdminBalance,
        "Admin balance did not increase after withdrawal"
      );
    } catch (error) {
      console.error("Error in withdraw test:", error);
      throw error;
    }
  });

```

Run the test. We can see that the treasury account SOL balance reduces and our balance increases.

![Screenshot showing the withdrawal from the treasury](https://r2media.rareskills.io/SolanaSPLTokenSale/image6.png)

### Test that non-admins cannot withdraw from the treasury

Add the following test block to confirm that unauthorized accounts cannot withdraw SOL from the treasury account balance.

An unauthorized account here, is any account whose public key does not match what we had previously stored in the `adminConfig` account during initialization.

```tsx

it("prevents non-admins from withdrawing funds", async () => {
    const nonAdminKeypair = web3.Keypair.generate();

    const amountToWithdraw = new anchor.BN(1e8);

    try {
      await program.methods
        .withdrawFunds(amountToWithdraw)
        .accounts({
          admin: nonAdminKeypair.publicKey,
          adminConfig: adminConfigPda,
          treasury: treasuryPda,
          systemProgram: anchor.web3.SystemProgram.programId,
        })
        .signers([nonAdminKeypair])
        .rpc();

      assert.fail("Non-admin was able to withdraw funds, but should be prohibited");
    } catch (error) {
      console.log("Expected error occurred:", error.toString().substring(0, 150) + "...");
      assert.include(
        error.toString(),
        "UnauthorizedAccess",
        "Expected unauthorized access error not received"
      );
      console.log("Non-admin withdrawal was correctly rejected");
    }
  });

```

Run the test.

![Screenshot showing that unauthorized access was prevented](https://r2media.rareskills.io/SolanaSPLTokenSale/image7.png)

From the logs we can see that Anchor throws an error when we try to withdraw funds with an unauthorized account.

## Summary

In this tutorial, we’ve demonstrated a practical use case of an SPL Token by building a Token Sale program (with a supply cap) that allows users to exchange SOL for our SPL token at a fixed rate.

The program utilizes two key PDAs: a self-referential mint PDA and a treasury PDA. We set the mint as its own authority with the `mint::authority = mint.key()` during initialization, thereby eliminating the need for a separate mint authority account. This pattern ensures that anybody can buy/mint the token through our program, without us having to authorize minting each time.

By using `CpiContext::new_with_signer` with properly derived seeds, our program can mint tokens to buyers and indirectly act as the mint authority.

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*