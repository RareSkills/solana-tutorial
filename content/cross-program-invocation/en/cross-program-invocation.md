# Cross Program Invocation In Anchor

Cross Program Invocation (CPI) is Solana's terminology for a program calling the public function of another program.

We've already done CPI before when we sent a [transfer SOL transaction to the system program](https://www.rareskills.io/post/anchor-transfer-sol). Here is the relevant snippet by way of reminder:

``` rust
pub fn send_sol(ctx: Context<SendSol>, amount: u64) -> Result<()> {  
  	let cpi_context = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: ctx.accounts.signer.to_account_info(),
            to: ctx.accounts.recipient.to_account_info(),
        }
    );

    let res = system_program::transfer(cpi_context, amount);

    if res.is_ok() {
        return Ok(());
    } else {
        return err!(Errors::TransferFailed);
    }
}
```

The `Cpi` in `CpiContext` literally stands for "Cross program invocation."

The workflow for calling the public functions of a program other than the System Program is not much different — and we will teach that in this tutorial.

This tutorial only focuses on how to call another Solana program that was built with Anchor. If the other program was developed with pure Rust, then the following guide will not work.

In our running example, the `Alice` program will call a function on the `Bob` program.

## The Bob Program
We start by creating a new project using Anchor's CLI:

```bash
anchor init bob
```
Then copy-paste the code below in `bob/lib.rs`. The account has two functions, one to initialize a storage account that holds a `u64` and a function `add_and_store` that takes two`u64`variables, adds them together, and stores them in the account defined by the struct `BobData`.

```rust

use anchor_lang::prelude::*;
use std::mem::size_of;

// REPLACE WITH YOUR <PROGRAM_ID>declare_id!("8GYu5JYsvAYoinbFTvW4AACYB5GxGstz21FmZe3MNFn4");

#[program]
pub mod bob {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        msg!("Data Account Initialized: {}", ctx.accounts.bob_data_account.key());

        Ok(())
    }

    pub fn add_and_store(ctx: Context<BobAddOp>, a: u64, b: u64) -> Result<()> {
        let result = a + b;
                        
        // MODIFY/UPDATE THE DATA ACCOUNT
        ctx.accounts.bob_data_account.result = result;
        Ok(())
    }
}

#[account]
pub struct BobData {
    pub result: u64,
}

#[derive(Accounts)]
pub struct BobAddOp<'info> {   
    #[account(mut)]
    pub bob_data_account: Account<'info, BobData>,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = size_of::<BobData>() + 8)]
    pub bob_data_account: Account<'info, BobData>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

The goal of this tutorial is to create another program `alice` that calls `bob.add_and_store`.


While still within the project (bob), create a new program using `anchor new` command:

```bash
anchor new alice
```

The command line should print out `Created new program`.

Before we start writing the program for Alice, the code snippet below has to be added to the **[dependencies]** section of the Alice's **Cargo.toml** file at `programs/alice/Cargo.toml`.

```rust
[dependencies]
bob = {path = "../bob", features = ["cpi"]}
```

Anchor is doing a significant amount of work in the background here. Alice now has access to the definition of Bob's public functions and Bob's structs. **You can think of this as being analogous to importing an interface in Solidity so that we know how to interact with another contract**.


Below we show the `Alice` program. At the top, the Alice program is importing the struct that carries the accounts for the `BobAddOp` (which is used for `add_and_store`). Pay attention to the comments in the code:

```rust

use anchor_lang::prelude::*;
// account struct for add_and_storeuse bob::cpi::accounts::BobAddOp;

// The program definition for Bob
use bob::program::Bob;

// the account where Bob is storing the sum
use bob::BobData;

declare_id!("6wZDNWprmb9TAZYMAPpT23kHDPABvBLT8jbWQKLHEmBy");

#[program]
pub mod alice {
    use super::*;

    pub fn ask_bob_to_add(ctx: Context<AliceOp>, a: u64, b: u64) -> Result<()> {
        let cpi_ctx = CpiContext::new(
            ctx.accounts.bob_program.to_account_info(),
            BobAddOp {
                bob_data_account: ctx.accounts.bob_data_account.to_account_info(),
            }
        );

        let res = bob::cpi::add_and_store(cpi_ctx, a, b);

        // return an error if the CPI failed
        if res.is_ok() {
            return Ok(());
        } else {
            return err!(Errors::CPIToBobFailed);
        }
    }
}

#[error_code]
pub enum Errors {
    #[msg("cpi to bob failed")]
    CPIToBobFailed,
}

#[derive(Accounts)]
pub struct AliceOp<'info> {
    #[account(mut)]
    pub bob_data_account: Account<'info, BobData>,

    pub bob_program: Program<'info, Bob>,
}
```
If we compare `ask_bob_to_add` to the code snippet at the top of this article where we showed how to transfer SOL, we see a lot of similarities.

![Cross Program Invocation](https://static.wixstatic.com/media/706568_107fea85400047e19d4653970968b609~mv2.jpeg/v1/fill/w_1480,h_540,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_107fea85400047e19d4653970968b609~mv2.jpeg)

To make a CPI, the following are required:
- A reference to the target program (as an `AccountInfo`) (<span style="color: red">red box</span>)
- The list of accounts needed by the function on the target program to run (the `ctx` struct which contains all the accounts) (<span style="color: green">green box</span>)
- The arguments to pass to the function (<span style="color: orange">orange box</span>)

## Testing The CPI
The following Typescript code can be used to test the CPI:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Bob } from "../target/types/bob";
import { Alice } from "../target/types/alice";
import { expect } from "chai";

describe("CPI from Alice to Bob", () => {
  const provider = anchor.AnchorProvider.env();

  // Configure the client to use the local cluster.
  anchor.setProvider(provider);

  const bobProgram = anchor.workspace.Bob as Program<Bob>;
  const aliceProgram = anchor.workspace.Alice as Program<Alice>;
  const dataAccountKeypair = anchor.web3.Keypair.generate();

  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await bobProgram.methods
      .initialize()
      .accounts({
        bobDataAccount: dataAccountKeypair.publicKey,
        signer: provider.wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([dataAccountKeypair])
      .rpc();
  });

  it("Can add numbers then double!", async () => {
    // Add your test here.
    const tx = await aliceProgram.methods
      .askBobToAddThenDouble(new anchor.BN(4), new anchor.BN(2))
      .accounts({
        bobDataAccount: dataAccountKeypair.publicKey,
        bobProgram: bobProgram.programId,
      })
      .rpc();
  });

  it("Can assert value in Bob's data account equals 4 + 2", async () => {

    const BobAccountValue = (
      await bobProgram.account.bobData.fetch(dataAccountKeypair.publicKey)    ).result.toNumber();
    expect(BobAccountValue).to.equal(6);
  });
});
```

## Doing CPI in one line
Because the ctx account passed to Alice contains a reference to all the accounts we need to conduct the transaction, we can create a function inside an `impl` for that struct that accomplish the CPI. Remember, all `impl` "attaches" functions to a struct that can use the data in the struct. Since the `ctx` struct `AliceOp` already holds all the accounts that `Bob` needs for the transaction, we can move all the CPI code:

```rust
let cpi_ctx = CpiContext::new(
    ctx.accounts.bob_program.to_account_info(),

    BobAddOp {
        bob_data_account: ctx.accounts.bob_data_account.to_account_info(),
    }
);
```

into an `impl` like so:

```rust
let cpi_ctx = CpiContext::new(
    ctx.accounts.bob_program.to_account_info(),
    BobAddOp {
        bob_data_account: ctx.accounts.bob_data_account.to_account_info(),
    }
);

use anchor_lang::prelude::*;
use bob::cpi::accounts::BobAddOp;
use bob::program::Bob;
use bob::BobData;

// REPLACE WITTH YOUR <PROGRAM_ID>declare_id!("B2BNs2GecG8Ux5EchDDFZakRWX2NDfy1RDhPCTJuJtr5");

#[program]
pub mod alice {
    use super::*;

    pub fn ask_bob_to_add(ctx: Context<AliceOp>, a: u64, b: u64) -> Result<()> {
        // Calls the `bob_add_operation` function in bob program
        let res = bob::cpi::bob_add_operation(ctx.accounts.add_function_ctx(), a, b);
        
        if res.is_ok() {
            return Ok(());
        } else {
            return err!(Errors::CPIToBobFailed);
        }
    }
}

impl<'info> AliceOp<'info> {
    pub fn add_function_ctx(&self) -> CpiContext<'_, '_, '_, 'info, BobAddOp<'info>> {
        // The bob program we are interacting with
        let cpi_program = self.bob_program.to_account_info();

        // Passing the necessary account(s) to the `BobAddOp` account struct in Bob program
        let cpi_account = BobAddOp {
            bob_data_account: self.bob_data_account.to_account_info(),
        };

        // Creates a `CpiContext` object using the new method
        CpiContext::new(cpi_program, cpi_account)
    }
}

#[error_code]
pub enum Errors {
    #[msg("cpi to bob failed")]
    CPIToBobFailed,
}

#[derive(Accounts)]
pub struct AliceOp<'info> {
    #[account(mut)]

    pub bob_data_account: Account<'info, BobData>,
    pub bob_program: Program<'info, Bob>,
}
```
We are able to make a CPI call to `Bob` in "one line." This could be handy if other parts of the Alice program made a CPI to Bob — moving the code to the `impl` would prevent us from copying and pasting the code to create the `CpiContext`.

## Learn more with RareSkills
This tutorial is part of a series on [learning Solana development](https://www.rareskills.io/solana-tutorial).

*Originally Published May, 17, 2024*