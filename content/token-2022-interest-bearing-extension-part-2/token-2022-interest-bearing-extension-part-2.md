# Interest Bearing Token Part 2

The interest-bearing extension adds the ability for a token mint to accrue interest over time. In our previous discussion of [Token-2022](https://rareskills.io/post/token-2022-interest-bearing-extension), we introduced the interest-bearing extension and explained how balances grow virtually without changing the raw on-chain account balance. Our focus was on how the extension works conceptually, and how Solana’s client functions calculate accrued interest.

In this article, we’ll put that knowledge into practice. We’ll build a management system using Anchor that creates an interest-bearing token mint programmatically under a [PDA (program-derived address](https://rareskills.io/post/solana-pda)) authority, which ensures only the program can control it. The system will also allow rate updates through a designated rate authority.

The program we’ll build will demonstrate the full lifecycle of an interest-bearing token: **initialization, minting, interest accrual, and rate changes.** We’ll also test interest accrual as time passes using time traveling with LiteSVM.

By the end of this article, you’ll have a solid understanding of how the interest-bearing token works in practice.

## Project initialization

We’ll start by creating a new Anchor project. Run the command below to initialize the project:

```bash
anchor init interest-bearing && cd interest-bearing
```

Now, update your  `program/src/Cargo.toml` file to include the `anchor-spl` dependency and enable `idl-build`. The `idl-build` feature makes Anchor generate IDL definitions for CPI (Cross-Program Invocation) calls, which we’ll use later when writing tests to call program functions.

```toml
[package]
name = "interest-bearing"
version = "0.1.0"
description = "Created with Anchor"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "interest_bearing"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"] # We added this

[dependencies]
anchor-lang = { version = "0.31.1", features = ["init-if-needed"] } # Include this 
anchor-spl = { version = "0.31.1", features = ["idl-build"] } # include this
```

You can now run `anchor build` successfully to confirm that your project is correctly setup.

## Project structure

The project will be in two phases:

1. The Anchor Rust program
2. And the TypeScript test

**1. The Anchor Rust program**

The Anchor Rust program will handle three core actions:

- Creating and initializing a new interest-bearing mint, and setting a PDA as the mint authority
- Minting interest bearing tokens
- Updating the interest rate through the rate authority.

Each of these actions will be implemented in Anchor as on-chain function entry points.

```rust
#[program]
pub mod interest_bearing {
    pub fn create_interest_bearing_mint(...) -> Result<()> { ... }
    pub fn mint_tokens(...) -> Result<()> { ... }
    pub fn update_rate(...) -> Result<()> { ... }
}
```

We’ll define the functions in the `programs/interest-bearing/src/lib.rs` file:

1. `create_interest_bearing_mint`: Creates a token mint with the `InterestBearingConfig` extension enabled and sets the rate authority.
2. `mint_tokens`: Mints tokens to a user’s account using a PDA as the mint authority.
3. `update_rate`: Updates the annual interest rate of the mint, restricted to the rate authority.

**2.   The TypeScript test**

The TypeScript test will verify that the program can:

- create an interest-bearing mint
- mint tokens under a PDA authority
- update the interest rate through the rate authority
- display virtual interest accrual accurately.

## **Implementing the Anchor Rust program**

Now that we understand the project structure, let’s implement the on-chain program itself.

We begin by importing the required Anchor and Token-2022 dependencies and declaring the program ID in the `program/interest-bearing/src/lib.rs` file:

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{self, CreateAccount};
use anchor_spl::{
    token_2022::{
        initialize_mint2,
        spl_token_2022::{
            extension::{ExtensionType},
            pod::PodMint,
        },
        InitializeMint2, Token2022,
    },
    token_interface::{Mint, TokenAccount, mint_to, MintTo},
    token_2022_extensions::interest_bearing_mint::{
        interest_bearing_mint_initialize,
        interest_bearing_mint_update_rate,
        InterestBearingMintInitialize,
        InterestBearingMintUpdateRate,
    },
};

declare_id!("5fRtYEKp81rfRRqcYxp9XMnucgvtt6JxQ5GDDQy2TMtx");
```

Notice that we didn’t install `spl-token-2022` directly—we’re using Anchor’s re-export instead. Mixing both can lead to version mismatches and runtime conflicts.

Finally, run `anchor keys sync` to ensure your program ID in the `declare_id!` macro matches the keypair defined in your `Anchor.toml`.

We have all the dependencies ready, now, let’s setup the workflow to create and initialize interest bearing token mints.

### i. Create and initialize interest bearing token mints

We’ll now create the `create_interest_bearing_mint` function:

```rust
pub fn create_interest_bearing_mint(...) -> Result<()> { ... }
```

The function performs four steps to set up a new Token-2022 mint with the `InterestBearingConfig` extension enabled. The steps are:

- **Step 1: Calculating the** `InterestBearingConfig` **account size**
- **Step 2:** **Creating the mint account and funding it with lamports for rent**
- **Step 3: Initialize the** `InterestBearingConfig` **extension**
- **Step 4: Running the standard `initialize_mint2` function**

**Step 1: Calculating the account size required**

When creating an account in Solana, you need to specify the size of the account and pay rent accordingly. 

We’ll use the `try_calculate_account_len` function from the `ExtensionType` we imported earlier to calculate the account size required to hold both the base mint data and the extension data automatically. This ensures the account is allocated with enough space for the `InterestBearingConfig` extension. 

```rust
let mint_size = ExtensionType::try_calculate_account_len::<Mint>(&[
    ExtensionType::InterestBearingConfig,
])?;
```

In part one, we discussed how to manually calculate the extension data, but we'll use `try_calculate_account_len` here. Using `try_calculate_account_len` is standard practice and allows us to calculate the accurate size of the Mint account and the extension data at once.

**Step 2: Create and fund the mint account**

Now that we have the mechanism to calculate the accurate size, we’ll create the mint account manually using the system program, funding it with rent-exempt lamports (**when an account holds enough lamports relative to its size, it becomes "rent-exempt" and will never be charged rent or deleted**). 

Anchor doesn’t perform this step automatically because Token-2022 mints require **custom sizing for extensions**. The `#[account(init)]` attribute on accounts assumes a fixed size (valid for standard SPL Token mints), but Token-2022 mints vary depending on which extensions they include. To handle this correctly, you must compute the required space yourself and create the account manually.

The code below creates the mint account with the exact space and lamports needed for rent exemption.

- `Rent::get()?.minimum_balance(mint_size)` computes the minimum lamports required to make the account rent-exempt based on its size.
- `system_program::create_account` then allocates and funds the account, assigning ownership to the Token-2022 program (`token_program.key()`).
- The CPI context specifies that the lamports come from the payer, and the new account being created is the mint.

This ensures the mint account is properly sized, rent-exempt, and owned by the correct program before any Token-2022 instructions initialize it.

```rust
// 2) Create the mint account with correct space and rent
 let lamports = Rent::get()?.minimum_balance(mint_size);
 system_program::create_account(
        CpiContext::new(
             ctx.accounts.system_program.to_account_info(),
             CreateAccount {
                 from: ctx.accounts.payer.to_account_info(),
                 to: ctx.accounts.mint.to_account_info(),
              },
        ),
        lamports,
        mint_size as u64,
        &ctx.accounts.token_program.key(),
  )?;
```

We’ll define the full `CreateInterestBearingMint` accounts struct that `&ctx` in the above code refers to later in this article.

**Step 3: Initialize the `InterestBearingConfig` extension**

Next, we initialize the `InterestBearingConfig` ****extension by setting the rate authority and the initial interest rate (in basis points).

This step must occur before ****initializing the base mint, since extensions must be set up first — otherwise, the mint’s layout won’t match the expected account size and `initialize_mint2` will fail.

```rust
    // 3) Initialize the interest-bearing extension BEFORE base mint init
    interest_bearing_mint_initialize(
        CpiContext::new(
           ctx.accounts.token_program.to_account_info(),
             InterestBearingMintInitialize {
               token_program_id: ctx.accounts.token_program.to_account_info(),
                 mint: ctx.accounts.mint.to_account_info(),
               },
        ),
        Some(ctx.accounts.rate_authority.key()),
        rate_bps,
    )?;
```

We used `Some(ctx.accounts.rate_authority.key())` here because the rate authority is optional. As mentioned in part one, if no rate authority is provided, the field will be populated with zeros, making the rate immutable.

**Step 4: Running the standard `initialize_mint2` function**

Finally, the code below initializes the base mint itself using the standard `initialize_mint2` CPI. This sets the mint’s decimals, assigns the PDA as the mint and freeze authority, and finalizes the Token-2022 mint configuration.

Because programs can’t hold private keys, the PDA acts as the mint’s authority. Whenever the program needs to sign on behalf of this PDA (for example, when minting new tokens), it must rederive the PDA using the same seed and bump combination (`[b"mint-authority", &[bump]]`).

Anchor exposes this bump via `ctx.bumps`.

The bump is a single-byte value (0–255) added during PDA derivation. It  ensures the resulting address can’t be generated from any private key. It must also be included in the signer seeds during PDA signature verification; otherwise, the verification will fail.

We also set both the **mint authority** and **freeze authority** to the PDA to ensure that only the program’s logic can mint or freeze tokens.

```
 // 4) Initialize base mint (decimals, authorities)
 let mint_auth_bump = ctx.bumps.mint_authority;
 let signer_seeds: &[&[&[u8]]] = &[&[b"mint-authority", &[mint_auth_bump]]];

 initialize_mint2(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                InitializeMint2 {
                    mint: ctx.accounts.mint.to_account_info(),
                },
                signer_seeds,
            ),
            decimals,
            &ctx.accounts.mint_authority.key(),
            Some(&ctx.accounts.mint_authority.key()),
        )?;
```

### **`CreateInterestBearingMint` accounts context**

Below is the struct that defines the accounts context for the `CreateInterestBearingMint` function we’ve used so far. 

Notice that the mint is declared as an `UncheckedAccount` instead of an `InterfaceAccount<Mint>` (an Anchor wrapper around `AccountInfo` that automatically validates an account to ensure it's an initialized token mint). 

We use `UncheckedAccount` here because we need to create the mint with extension space, and Anchor can’t validate it as a `Mint` until after initialization.

The struct defines `mint_authority` PDA with its seed and bump. Once that’s done, the program logic can mint or freeze tokens, but no external keypair can.

The struct also defines other accounts we’ve used; we’ve added comments specifying them.

```rust
#[derive(Accounts)]
pub struct CreateInterestBearingMint<'info> {
    /// CHECK: This account is created manually as a Token-2022 mint with extensions.
    #[account(mut)]
    pub payer: Signer<'info>,

    /// CHECK: PDA account used as mint and freeze authority
    #[account(
        seeds = [b"mint-authority"],
        bump
    )]
    pub mint_authority: UncheckedAccount<'info>,

    /// Raw mint account to be created with extension space
    /// CHECK: We trust the token program to validate this is a proper mint account.
    #[account(mut, signer)]
    pub mint: UncheckedAccount<'info>,

    /// Token-2022 program
    pub token_program: Program<'info, Token2022>,

    pub system_program: Program<'info, System>,
    /// Signer that will control interest rate updates
    pub rate_authority: Signer<'info>,
}
```

The complete code for creating the token mint we’ve discussed so far is shown below:

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{self, CreateAccount};
use anchor_spl::{
    token_2022::{
        initialize_mint2,
        spl_token_2022::{
            extension::{ExtensionType},
            pod::PodMint,
        },
        InitializeMint2, Token2022,
    },
    token_interface::{Mint, TokenAccount, mint_to, MintTo},
    token_2022_extensions::interest_bearing_mint::{
        interest_bearing_mint_initialize,
        interest_bearing_mint_update_rate,
        InterestBearingMintInitialize,
        InterestBearingMintUpdateRate,
    },
};

declare_id!("5fRtYEKp81rfRRqcYxp9XMnucgvtt6JxQ5GDDQy2TMtx");

#[program]
pub mod interest_bearing {
    use super::*;

    pub fn create_interest_bearing_mint(
        ctx: Context<CreateInterestBearingMint>,
        rate_bps: i16,
        decimals: u8,
    ) -> Result<()> {
        msg!("Create interest-bearing mint @ {} bps", rate_bps);

        // 1) Compute mint size including extension header + InterestBearingConfig
        let mint_size = ExtensionType::try_calculate_account_len::<PodMint>(&[
            ExtensionType::InterestBearingConfig,
        ])?;

        // 2) Create the mint account with correct space and rent
        let lamports = Rent::get()?.minimum_balance(mint_size);
        system_program::create_account(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                CreateAccount {
                    from: ctx.accounts.payer.to_account_info(),
                    to: ctx.accounts.mint.to_account_info(),
                },
            ),
            lamports,
            mint_size as u64,
            &ctx.accounts.token_program.key(),
        )?;

        // 3) Initialize the interest-bearing extension BEFORE base mint init
        interest_bearing_mint_initialize(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                InterestBearingMintInitialize {
                    token_program_id: ctx.accounts.token_program.to_account_info(),
                    mint: ctx.accounts.mint.to_account_info(),
                },
            ),
            Some(ctx.accounts.rate_authority.key()),
            rate_bps,
        )?;

        // 4) Initialize base mint (decimals, authorities)
        let mint_auth_bump = ctx.bumps.mint_authority;
        let signer_seeds: &[&[&[u8]]] = &[&[b"mint-authority", &[mint_auth_bump]]];

        initialize_mint2(
            CpiContext::new_with_signer(
                ctx.accounts.token_program.to_account_info(),
                InitializeMint2 {
                    mint: ctx.accounts.mint.to_account_info(),
                },
                signer_seeds,
            ),
            decimals,
            &ctx.accounts.mint_authority.key(),
            Some(&ctx.accounts.mint_authority.key()),
        )?;

        Ok(())
    }
    
		#[derive(Accounts)]
		pub struct CreateInterestBearingMint<'info> {
		    /// CHECK: This account is created manually as a Token-2022 mint with extensions.
		    #[account(mut)]
		    pub payer: Signer<'info>,
		
		    /// CHECK: PDA account used as mint and freeze authority
		    #[account(
		        seeds = [b"mint-authority"],
		        bump
		    )]
		    pub mint_authority: UncheckedAccount<'info>,
		
		    /// Raw mint account to be created with extension space
		    /// CHECK: We trust the token program to validate this is a proper mint account.
		    // #[account(mut)]
		    #[account(mut, signer)]
		    pub mint: UncheckedAccount<'info>,
		
		    /// Token-2022 program
		    pub token_program: Program<'info, Token2022>,
		
		    pub system_program: Program<'info, System>,
		    /// Signer that will control interest rate updates
		    pub rate_authority: Signer<'info>,
		}
}
```

Now that we’ve created the token mints and initialized the extension, let’s proceed to implementing the `mint_tokens` function.

## ii. Creating the Mint tokens function

The `mint_tokens` function mints tokens to a user’s account, using the PDA as the mint authority. 

Here is what the `mint_tokens` function does:  

- It first retrieves the PDA’s bump and reconstructs the signer seeds needed for verification.
- Then, it calls the Token-2022 program’s `mint_to` CPI. It passes the signer seeds via `CpiContext::new_with_signer`, the runtime recognizes the PDA as the authorized signer and mints the specified number of tokens to the recipient’s token account.

```rust
pub fn mint_tokens(ctx: Context<MintTokens>, amount: u64) -> Result<()> {

    // Fetch the bump for the PDA so we can recreate the same signer seeds
    let bump = ctx.bumps.mint_authority;
    let signer_seeds: &[&[&[u8]]] = &[&[b"mint-authority", &[bump]]];

    // Call into the Token-2022 program to mint tokens
    // `CpiContext::new_with_signer` lets us pass the PDA seeds so the runtime
    // can treat the PDA as if it signed the instruction
    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                // Mint account whose supply will increase
                mint: ctx.accounts.mint.to_account_info(),
                // Recipient’s token account that will receive the minted tokens
                to: ctx.accounts.to_token_account.to_account_info(),
                // PDA that acts as mint authority
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            signer_seeds,
        ),
        amount, // Number of tokens to mint
    )?;

    Ok(())
}

```

Below is the struct that defines all the accounts we used in the above `mint_tokens` function. 
It lists all accounts needed to mint new tokens under the PDA’s authority, ensuring the correct mint, recipient, and Token-2022 program are used.

```rust
pub struct MintTokens<'info> {
    /// CHECK: PDA authority must match the seed used during mint init
    #[account(
        seeds = [b"mint-authority"],
        bump
    )]
    /// CHECK: This is the mint authority PDA we created during mint init.
    pub mint_authority: UncheckedAccount<'info>,

    /// Use token_interface to bind this Mint to Token2022 program
    /// CHECK: We trust the token program to validate this is a proper mint account.
    #[account(mut, mint::token_program = token_program)]
    pub mint: InterfaceAccount<'info, Mint>,

    #[account(mut, token::mint = mint, token::authority = recipient)]
    pub to_token_account: InterfaceAccount<'info, TokenAccount>,

    pub recipient: Signer<'info>,
    pub token_program: Program<'info, Token2022>,
}
```

## **iii. Updating the interest rate**

We’ve seen how to create the token mint, initialize the extension, and mint tokens. The next step is updating the interest rate.

The `update_rate` function below ensures only the configured `rate_authority` can update the mint’s annual interest rate. It does that by calling the Token-2022 CPI `interest_bearing_mint_update_rate`. 

The function also uses the `InterestBearingMintUpdateRate` struct to specify which accounts (mint, token program, and rate authority) are required for the CPI call, then verifies that the signer matches the configured authority internally before updating the rate stored in the mint’s extension data.

```rust
    pub fn update_rate(ctx: Context<UpdateRate>, new_rate_bps: i16) -> Result<()> {
        msg!("Update interest rate -> {} bps", new_rate_bps);

        // Call into the Token-2022 program to update interest rate on the mint
        // The CPI will check that the provided rate_authority signer matches the
        // authority configured in the mint's extension data
        interest_bearing_mint_update_rate(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                InterestBearingMintUpdateRate {
                    token_program_id: ctx.accounts.token_program.to_account_info(),
                    mint: ctx.accounts.mint.to_account_info(),
                    rate_authority: ctx.accounts.rate_authority.to_account_info(),
                },
            ),
            new_rate_bps, // new interest rate in basis points (1% = 100 bps)
        )?;

        Ok(())
    }

```

The context struct below defines the accounts required to update the rate. The `rate_authority` must sign the transaction, and the Token-2022 program ensures it matches the authority set in the mint’s extension.

```rust
#[derive(Accounts)]
pub struct UpdateRate<'info> {
    /// CHECK: This is the mint account we’re updating. We rely on Token-2022
    /// program logic to validate its data, so Anchor does not need to enforce checks here.
    #[account(mut, mint::token_program = token_program)]
    pub mint: InterfaceAccount<'info, Mint>,

    /// Must sign and match the extension’s configured rate authority
    pub rate_authority: Signer<'info>,

    pub token_program: Program<'info, Token2022>,
}

```

### Complete implementation

You can clone the full implementation from the repository below to explore the complete code:

```bash
git clone [https://github.com/ezesundayeze/interest-bearing-mint](https://github.com/ezesundayeze/interest-bearing-mint/blob/main/programs/interest-bearing/src/lib.rs)
```

## The TypeScript test

First, build the program by running `anchor build` on your Terminal.

We’ll need to test different timelines to truly demonstrate yields accrual over a period of time. This can be tricky to do in tests, we’ll use [LiteSVM](https://rareskills.io/post/litesvm)—a lightweight Solana virtual machine to simulate time progression and verify interest accrual without running on a live cluster.

Install LiteSVM and the Solana SPL token library dependency we’ll use to interact with the token:

```bash
yarn add anchor-litesvm @solana/spl-token
```

Replace the contents of your `interest_bearing.ts` file with the code below. 

This test interacts with our program to demonstrates how the Interest-Bearing Token extension compounds value over time.  

The test follows these steps:

1. **Initialize an interest-bearing mint:** Creates a new token mint configured with a 3% annual interest rate and assigns an authority that can later update this rate. The initialization timestamp is also recorded for precise interest tracking.
2. **Mint tokens to a recipient:** Mints 1000 tokens into the recipient’s associated token account. The test confirms that the on-chain mint configuration and token balances are correct at initialization.
3. **Simulate compounding over multiple periods:**
    
    Uses LiteSVM’s virtual clock to fast-forward time and demonstrate compounding growth:
    
    - **Period 1:** 3 months at 3% annual rate
    - **Period 2:** 9 months more at 5% annual rate (after rate update, at the 12th month)
    - **Period 3:** 3 months more at 7% annual rate (final period, the 15th month)
    
    Each period calculates:
    
    - The expected balance using the continuous compounding formula $A = P \times e^{r t}$
    - The virtual balance computed by the SPL Token helpers, which reflect accrued interest.
    
    Both results are compared to confirm that the virtual growth matches the mathematical expectation of continuous compounding.
    
4. **Validate compound growth over 15 months:**
    
    Confirms that the token balance grows in line with the expected exponential curve, even after multiple rate changes. The test also prints intermediate results to show how compounding evolves at each stage.
    
    This code contains comments that explains what each block is doing:
    

```tsx
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { InterestBearing } from "../target/types/interest_bearing";
import {
  TOKEN_2022_PROGRAM_ID,
  createAssociatedTokenAccountInstruction,
  getAssociatedTokenAddressSync,
  getAccount,
  getMint,
  getInterestBearingMintConfigState,
} from "@solana/spl-token";
import {
  PublicKey,
  Keypair,
  Transaction,
  LAMPORTS_PER_SOL,
} from "@solana/web3.js";
import { fromWorkspace, LiteSVMProvider } from "anchor-litesvm";
import assert from "assert";

// Constants for interest calculations (must be at module level)
const SECONDS_PER_YEAR = 365.24 * 24 * 60 * 60; // 365.24 days
const ONE_IN_BASIS_POINTS = 10000;

/**
 * Calculate the exponential factor for continuous compounding
 * This mirrors the SPL Token implementation exactly.
 * We are copying it here because it's not exported from the SPL token library.
 *
 * Formula: e^((rate * timespan) / (SECONDS_PER_YEAR * 10000))
 *
 * @param t1 - Start time in seconds
 * @param t2 - End time in seconds
 * @param rateBps - Interest rate in basis points
 */
const calculateExponentForTimesAndRate = (
  t1: number,
  t2: number,
  rateBps: number
): number => {
  const timespan = t2 - t1;
  const numerator = rateBps * timespan;
  const exponent = numerator / (SECONDS_PER_YEAR * ONE_IN_BASIS_POINTS);
  return Math.exp(exponent);
};

describe("interest-bearing", () => {
  // Set up a lightweight Solana VM for testing
  const svm = fromWorkspace("./").withBuiltins().withSysvars();
  const provider = new LiteSVMProvider(svm);
  anchor.setProvider(provider);

  // Get reference to our compiled program
  const program = anchor.workspace.InterestBearing as Program<InterestBearing>;

  // Key accounts we'll use throughout the tests
  let mint: Keypair;
  let rateAuthority: Keypair;
  let recipient: Keypair;
  let recipientAta: PublicKey;

  // Interest rates in basis points (1 basis point = 0.01%)
  const RATE_1_BPS = 300; // 3.00% annual rate
  const RATE_2_BPS = 500; // 5.00% annual rate
  const RATE_3_BPS = 700; // 7.00% annual rate

  // More precise year definition (accounts for leap years)
  const SECONDS_PER_YEAR = 365.24 * 24 * 60 * 60; // ~31,556,736 seconds

  // Token configuration
  const DECIMALS = 9;
  const INITIAL_BALANCE = 1000; // Start with 1000 tokens (UI amount)

  // Starting point for our virtual clock (Jan 1, 2024)
  const INITIAL_TIMESTAMP = 1704067200n;

  /**
   * Get UI amount for interest-bearing tokens
   * This implements the exact same logic as amountToUiAmountForInterestBearingMintWithoutSimulation
   * from the SPL Token library, adapted for LiteSVM
   *
   * The calculation happens in two phases:
   * 1. Pre-update: Interest from initialization to last rate update
   * 2. Post-update: Interest from last rate update to current time
   *
   * Total scale = e^(r1*t1) * e^(r2*t2)
   */
  const getInterestBearingUiAmount = async (
    rawAmount: bigint
  ): Promise<number> => {
    // Fetch mint configuration
    const mintInfo = await getMint(
      provider.connection,
      mint.publicKey,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    const interestConfig = getInterestBearingMintConfigState(mintInfo);
    if (!interestConfig) {
      throw new Error("Interest config not found");
    }

    // Get current timestamp from LiteSVM clock
    const currentTimestamp = Number(svm.getClock().unixTimestamp);
    const lastUpdateTimestamp = Number(interestConfig.lastUpdateTimestamp);
    const initializationTimestamp = Number(
      interestConfig.initializationTimestamp
    );

    // Calculate pre-update exponent (initialization to last update)
    const preUpdateExp = calculateExponentForTimesAndRate(
      initializationTimestamp,
      lastUpdateTimestamp,
      interestConfig.preUpdateAverageRate
    );

    // Calculate post-update exponent (last update to current time)
    const postUpdateExp = calculateExponentForTimesAndRate(
      lastUpdateTimestamp,
      currentTimestamp,
      interestConfig.currentRate
    );

    // Total scale factor is the product of both exponentials
    const totalScale = preUpdateExp * postUpdateExp;

    // Apply the scale to the raw amount
    const scaledAmount = Number(rawAmount) * totalScale;

    // Convert to UI amount by dividing by decimal factor
    const decimalFactor = Math.pow(10, DECIMALS);
    const uiAmount = Math.trunc(scaledAmount) / decimalFactor;

    return uiAmount;
  };

  /**
   * Manually calculate expected balance with continuous compounding
   * This serves as our "test oracle" to verify the SPL Token calculations are correct
   *
   * Formula: A_final = A_start * e^(rate * time_in_years)
   */
  const calculateExpectedBalance = (
    startBalance: number,
    rateBps: number,
    timeInYears: number
  ): number => {
    const rateDecimal = rateBps / 10000;
    return startBalance * Math.exp(rateDecimal * timeInYears);
  };

  before(async () => {
    // Set our virtual clock to Jan 1, 2024 (for a consistent starting point)
    const clock = svm.getClock();
    clock.unixTimestamp = INITIAL_TIMESTAMP;
    svm.setClock(clock);
    console.log("Initial clock set to:", INITIAL_TIMESTAMP.toString());

    // Generate fresh keypairs for this test run
    mint = Keypair.generate();
    rateAuthority = Keypair.generate();
    recipient = Keypair.generate();

    // Give accounts some SOL to pay for transactions
    svm.airdrop(provider.wallet.publicKey, BigInt(5 * LAMPORTS_PER_SOL));
    svm.airdrop(recipient.publicKey, BigInt(5 * LAMPORTS_PER_SOL));
  });

  /**
   * Test 1: Create the interest-bearing mint
   */
  it("creates an interest bearing mint", async () => {
    // Call our program to initialize the mint with starting rate of 3%
    await program.methods
      .createInterestBearingMint(RATE_1_BPS, DECIMALS)
      .accounts({
        payer: provider.wallet.publicKey, // Who pays for the transaction
        mint: mint.publicKey, // The new mint we're creating
        rateAuthority: rateAuthority.publicKey, // Who can update interest rates
      })
      .signers([rateAuthority, mint])
      .rpc();

    // Verify the mint was created with correct configuration
    const mintInfo = await getMint(
      provider.connection,
      mint.publicKey,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    const interestConfig = await getInterestBearingMintConfigState(mintInfo);
    console.log("Interest-bearing config:", {
      rateAuthority: interestConfig?.rateAuthority?.toBase58(),
      currentRate: interestConfig?.currentRate,
      initializationTimestamp: interestConfig?.initializationTimestamp,
      lastUpdateTimestamp: interestConfig?.lastUpdateTimestamp,
    });

    // Ensure initialization timestamp was recorded (important for interest calculations)
    assert.ok(
      interestConfig?.initializationTimestamp !== 0,
      "Initialization timestamp should not be 0"
    );
  });

  /**
   * Test 2: Mint initial tokens to recipient
   */
  it("mints tokens to a recipient", async () => {
    recipientAta = getAssociatedTokenAddressSync(
      mint.publicKey,
      recipient.publicKey,
      false,
      TOKEN_2022_PROGRAM_ID
    );

    // Create the ATA (it doesn't exist yet)
    const createAtaTx = new Transaction().add(
      createAssociatedTokenAccountInstruction(
        provider.wallet.publicKey,
        recipientAta,
        recipient.publicKey,
        mint.publicKey,
        TOKEN_2022_PROGRAM_ID
      )
    );
    await provider.sendAndConfirm(createAtaTx, []);

    // Mint the initial balance of tokens to the recipient
    // Convert UI amount (1000) to raw amount (1000 * 10^9)
    await program.methods
      .mintTokens(new anchor.BN(INITIAL_BALANCE * 10 ** DECIMALS))
      .accounts({
        mint: mint.publicKey,
        toTokenAccount: recipientAta,
        recipient: recipient.publicKey,
      })
      .signers([recipient])
      .rpc();

    // Verify the correct amount was minted
    const tokenAccount = await getAccount(
      provider.connection,
      recipientAta,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    // For interest-bearing tokens, we need to use the SPL Token method to get UI amount
    const balance = await getInterestBearingUiAmount(tokenAccount.amount);
    assert.strictEqual(
      balance,
      INITIAL_BALANCE,
      `Initial balance should be ${INITIAL_BALANCE}`
    );
    console.log(`Initial balance: ${balance} tokens`);
  });

  /**
   * Test 3: demonstrate compound interest over 15 months
   *
   * Timeline:
   * 1. Start with 1000 tokens at 3% rate
   * 2. Wait 3 months → balance grows with 3% rate
   * 3. Change rate to 5%
   * 4. Wait 9 more months → balance grows with 5% rate (12 months total)
   * 5. Change rate to 7%
   * 6. Wait 3 more months → balance grows with 7% rate (15 months total)
   */
  it("demonstrates compounded interest growth: 3 months, 12 months, 15 months", async () => {
    console.log("\n=== Starting Interest Accrual Test ===");
    console.log(`Starting balance: ${INITIAL_BALANCE} tokens\n`);

    // ==================================
    // PERIOD 1: First 3 months at 3% annual rate
    // ==================================
    console.log(`\n--- Period 1: 3 Months @ ${RATE_1_BPS / 100}% ---`);

    // Fast-forward time by 3 months (0.25 years)
    const clock1 = svm.getClock();
    clock1.unixTimestamp += BigInt(Math.floor(SECONDS_PER_YEAR * 0.25));
    svm.setClock(clock1);

    // Check the recipient's token balance
    const tokenAccount1 = await getAccount(
      provider.connection,
      recipientAta,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    // Use the official SPL Token method to get UI amount with interest applied
    const balanceAfter3Months = await getInterestBearingUiAmount(
      tokenAccount1.amount
    );

    // Calculate what we expect using the continuous compounding formula
    const expectedBalance1 = calculateExpectedBalance(
      INITIAL_BALANCE,
      RATE_1_BPS,
      0.25
    );

    console.log(`Balance after 3 months: ${balanceAfter3Months.toFixed(6)}`);
    console.log(
      `Expected balance (A = P e^{r t}): ${expectedBalance1.toFixed(6)}`
    );
    console.log(
      `Interest earned: ${(balanceAfter3Months - INITIAL_BALANCE).toFixed(6)}`
    );

    // Verify the calculation is correct (within 0.01 token tolerance)
    assert.ok(
      Math.abs(balanceAfter3Months - expectedBalance1) < 0.01,
      "Balance after 3 months is incorrect"
    );

    // ===============================================
    // PERIOD 2: Change rate to 5%, then advance 9 more months
    // ===============================================

    // Update the interest rate to 5%
    await program.methods
      .updateRate(RATE_2_BPS)
      .accounts({
        mint: mint.publicKey,
        rateAuthority: rateAuthority.publicKey,
      })
      .signers([rateAuthority])
      .rpc();

    console.log(
      `\n--- Period 2: 9 Months @ ${
        RATE_2_BPS / 100
      }% after initial 3 months (total = 12 months) ---`
    );

    // Fast-forward time by 9 more months (total of 12 months from start)
    const clock2 = svm.getClock();
    clock2.unixTimestamp += BigInt(Math.floor(SECONDS_PER_YEAR * 0.75));
    svm.setClock(clock2);

    const tokenAccount2 = await getAccount(
      provider.connection,
      recipientAta,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    // Get UI amount using SPL Token's official method
    const balanceAfter12Months = await getInterestBearingUiAmount(
      tokenAccount2.amount
    );

    // Expected: (balance after 3 months) * e^(0.05 * 0.75)
    const expectedBalance2 = calculateExpectedBalance(
      balanceAfter3Months,
      RATE_2_BPS,
      0.75
    );

    console.log(`Balance after 12 months: ${balanceAfter12Months.toFixed(6)}`);
    console.log(
      `Expected balance (A2 = A1 * e^{r2 * 0.75}): ${expectedBalance2.toFixed(
        6
      )}`
    );
    console.log(
      `Total interest earned: ${(
        balanceAfter12Months - INITIAL_BALANCE
      ).toFixed(6)}`
    );

    assert.ok(
      Math.abs(balanceAfter12Months - expectedBalance2) < 0.01,
      "Balance after 12 months is incorrect"
    );

    // ==============================================
    // PERIOD 3: Change rate to 7%, then advance final 3 months
    // ==============================================

    // Update the interest rate to 7%
    await program.methods
      .updateRate(RATE_3_BPS)
      .accounts({
        mint: mint.publicKey,
        rateAuthority: rateAuthority.publicKey,
      })
      .signers([rateAuthority])
      .rpc();

    console.log(
      `\n--- Period 3: extra 3 Months @ ${
        RATE_3_BPS / 100
      }% (total = 15 months) ---`
    );

    // Fast-forward time by 3 final months (total of 15 months from start)
    const clock3 = svm.getClock();
    clock3.unixTimestamp += BigInt(Math.floor(SECONDS_PER_YEAR * 0.25));
    svm.setClock(clock3);

    const tokenAccount3 = await getAccount(
      provider.connection,
      recipientAta,
      "confirmed",
      TOKEN_2022_PROGRAM_ID
    );

    // Get final UI amount using SPL Token's official method
    const balanceAfter15Months = await getInterestBearingUiAmount(
      tokenAccount3.amount
    );

    // Expected: (balance after 12 months) * e^(0.07 * 0.25)
    const expectedBalance3 = calculateExpectedBalance(
      balanceAfter12Months,
      RATE_3_BPS,
      0.25
    );

    console.log(`Balance after 15 months: ${balanceAfter15Months.toFixed(6)}`);
    console.log(
      `Expected balance (A3 = A2 * e^{r3 * 0.25}): ${expectedBalance3.toFixed(
        6
      )}`
    );
    console.log(
      `Total interest earned: ${(
        balanceAfter15Months - INITIAL_BALANCE
      ).toFixed(6)}`
    );
    console.log(
      `Effective return over 15 months: ${(
        (balanceAfter15Months / INITIAL_BALANCE - 1) *
        100
      ).toFixed(6)}%`
    );

    // Final verification (slightly larger tolerance for accumulated rounding)
    assert.ok(
      Math.abs(balanceAfter15Months - expectedBalance3) < 0.02,
      "Final balance after 15 months is incorrect"
    );
  });
});

```

Run the test with the command `anchor test`. The test output should look like this:

![A screenshot of a command-line terminal showing a successful interest accrual test. The test begins with a 1000 token balance and simulates three periods: 1) 3 months at 3% interest, 2) another 9 months at 5%, and 3) a final 3 months at 7%. Each step shows the new balance, the expected balance calculated with the formula A=Pert
, and the total interest earned, confirming the logic is correct.](https://r2media.rareskills.io/SolanaInterestBearingTokensPart2/image1.png)

From the above screenshot, you’ll notice that our interest accrual works correctly and aligns with the continuous compounding interest calculation we discussed earlier.

## Conclusion

So far we’ve gone through the complete life cycle of an interest bearing extension. This allowed us to understand more deeply how the interest bearing extension works beyond the concepts. And gives you a concrete starting point for experimenting with extensions and integrating them into real programs.

## Self-Study Exercise

### Build a Simple Staking Rewards Program

Users deposit regular tokens (like USDC) into a staking pool and receive interest bearing "receipt tokens" that automatically grow in value over time, eliminating the need for complex reward claiming mechanisms.

**Tag @rareskills_io on X when you have successfully built out your prototype!**

*This article is part of a [tutorial series on Solana](https://rareskills.io/solana-tutorial).*
