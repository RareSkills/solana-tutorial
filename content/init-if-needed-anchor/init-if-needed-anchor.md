# Init_if_needed in Anchor and the Reinitialization Attack

![Hero image showing Anchor init_if_needed](https://static.wixstatic.com/media/935a00_da6446a3727044b589537f2fedac8c55~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_da6446a3727044b589537f2fedac8c55~mv2.jpg)

In previous tutorials, we've had to initialize an account in a separate transaction before we can write data to it. We may wish to be able to initialize an account and write data to it in one transaction to simplify things for the user.

Anchor provides a handy macro called `init_if_needed` which, as the name suggests, will initialize the account if it does not exist.

The example counter below does not need a separate initialize transaction, it will start adding "1" to the `counter` storage right away.

Rust:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("9DbiqCqtqgP3NYufxBakbeRd7JyNpNYbsm6Jqrn8Z2Hn");

#[program]
pub mod init_if_needed {
    use super::*;

    pub fn increment(ctx: Context<Initialize>) -> Result<()> {
        let current_counter = ctx.accounts.my_pda.counter;
        ctx.accounts.my_pda.counter = current_counter + 1;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init_if_needed,
        payer = signer,
        space = size_of::<MyPDA>() + 8,
        seeds = [],
        bump
    )]
    pub my_pda: Account<'info, MyPDA>,

    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyPDA {
    pub counter: u64,
}
```

Typescript:


```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { InitIfNeeded } from "../target/types/init_if_needed";

describe("init_if_needed", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.InitIfNeeded as Program<InitIfNeeded>;

  it("Is initialized!", async () => {
    const [myPda, _bump] = anchor.web3.PublicKey.findProgramAddressSync([], program.programId);
    await program.methods.increment().accounts({myPda: myPda}).rpc();
    await program.methods.increment().accounts({myPda: myPda}).rpc();
    await program.methods.increment().accounts({myPda: myPda}).rpc();

    let result = await program.account.myPda.fetch(myPda);
    console.log(`counter is ${result.counter}`);
  });
});
```

When we try to build this program with `anchor build`, we will get the following error:

![error: init_if_needed](https://static.wixstatic.com/media/935a00_c444fda7ad5a4771ac4444d8993eb1f7~mv2.png/v1/fill/w_1480,h_216,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_c444fda7ad5a4771ac4444d8993eb1f7~mv2.png)

To make the error `init_if_needed requires that anchor-lang be imported with the init-if-needed cargo feature enabled` go away, we can open the `Cargo.toml` file in `programs/<anchor_project_name>` and add the following line:

```toml
[dependencies]
anchor-lang = { version = "0.29.0", features = ["init-if-needed"] }
```

But before we just silence the error, we should understand what a re-initialization attack is and how it can occur.

## In Anchor programs, accounts cannot be initialized twice (by default)
If we try to initialize an account that has already been initialized, the transaction will fail.

## How does Anchor know an account is already initialized?
From Anchor's perspective, if the account has a non-zero lamport balance OR the account is owned by the system program, then it is not initialized.

An account owned by the system program or with zero lamport balance can be initialized again.

To illustrate this, we have a Solana program with the typical `initialize` function (which uses `init`, not `init_if_needed`). It also has a `drain_lamports` function and a `give_to_system_program` function, both of which do what their names suggest:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;
use anchor_lang::system_program;

declare_id!("FC467mPCCKXG97ut1WdLLi55vuAcyCW8AD1vid27bZfn");

#[program]
pub mod reinit_attack {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn drain_lamports(ctx: Context<DrainLamports>) -> Result<()> {
        let lamports = ctx.accounts.my_pda.to_account_info().lamports();
        ctx.accounts.my_pda.sub_lamports(lamports)?;
				ctx.accounts.signer.add_lamports(lamports)?;
        Ok(())
    }

    pub fn give_to_system_program(ctx: Context<GiveToSystemProgram>) -> Result<()> {
        let account_info = &mut ctx.accounts.my_pda.to_account_info();
        // the assign method changes the owner
				account_info.assign(&system_program::ID);
        account_info.realloc(0, false)?;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct DrainLamports<'info> {
    #[account(mut)]
    pub my_pda: Account<'info, MyPDA>,
    #[account(mut)]
    pub signer: Signer<'info>,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8, seeds = [], bump)]
    pub my_pda: Account<'info, MyPDA>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct GiveToSystemProgram<'info> {
    #[account(mut)]
    pub my_pda: Account<'info, MyPDA>,
}

#[account]
pub struct MyPDA {}
```

Now consider the following unit test:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ReinitAttack } from "../target/types/reinit_attack";

describe("Program", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ReinitAttack as Program<ReinitAttack>;

  it("initialize after giving to system program or draining lamports", async () => {
    const [myPda, _bump] = anchor.web3.PublicKey.findProgramAddressSync([], program.programId);

    await program.methods.initialize().accounts({myPda: myPda}).rpc();

    await program.methods.giveToSystemProgram().accounts({myPda: myPda}).rpc();

    await program.methods.initialize().accounts({myPda: myPda}).rpc();
    console.log("account initialized after giving to system program!")

    await program.methods.drainLamports().accounts({myPda: myPda}).rpc();

    await program.methods.initialize().accounts({myPda: myPda}).rpc();
    console.log("account initialized after draining lamports!")
  });
});
```

The sequence is as follows:
1. We initialize the PDA
2. We transfer ownership of the PDA to the system program
3. We call initialize again, and it succeeds
4. We empty the lamports from the `my_pda` account
5. With zero lamport balance, the Solana runtime considers the account non-existent as it will be scheduled for deletion as it is no longer rent exempt.
6. We call initialize again, and it succeeds.**We have successfully reinitialized the account after following this sequence.**

Again, Solana does not have an "initialized" flag or anything. Anchor will allow an initialize transaction to succeed if the owner is the system program or the lamport balance is zero.

## Why reinitialization might be a problem in our example
Transferring ownership to the system program requires erasing the data in the account. Removing all the lamports "communicates" that you don't want the account to continue to exist.

Is your intent by doing either of those actions to restart the counter or end the life of the counter? If your application never expects the counter to be reset, this could lead to bugs.

Anchor wants you to think through your intent with this, which is why it makes you jump through the extra hoop of enabling a feature flag in Cargo.toml.

If you are okay with the counter getting reset back at some point and counting back up, reinitialization is not an issue. But if the counter should never reset to zero under any circumstance, then it would probably be better for you to implement the `initialization` function separately and add a safeguard to make sure it can only be called once in it's lifetime (for example, storing a boolean flag in a separate account).

Of course, your program might not necessarily have the mechanism to transfer the account to the system program or withdraw lamports from the account. But Anchor has no way of knowing this, so it always throws out the warning about `init_if_needed` because it cannot determine whether the account can go back into an initializable state.

## Having two initialization paths could lead to an off-by-one error or other surprising behavior
In our counter example with `init_if_needed`, the counter is never equal to zero because the first initialization transaction also increments the value from zero to one.

If we *also* had a regular initialization function that did not increment the counter, then the counter would be initialized and have a value of zero. If some business logic never expects to see a counter with a value of zero, then unexpected behavior may happen.

**In Ethereum, storage values for variables that have never been "touched" have a default value of zero. In Solana, accounts that have not been initialized do not hold zero-value variables — they don't exist and cannot be read.**

## "Initialization" does not always mean "init" in Anchor
Somewhat confusingly, some use the term "initialize" to mean "writing data the account for the first time" in a more general sense than Anchor's `init` macro.

If we look at the example program from [Soldev](https://solana.com/developers/courses), we see that the `init` macro isn't used:

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod initialization_insecure {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let mut user = User::try_from_slice(&ctx.accounts.user.data.borrow()).unwrap();
        user.authority = ctx.accounts.authority.key();
        user.serialize(&mut *ctx.accounts.user.data.borrow_mut())?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    user: AccountInfo<'info>,
    #[account(mut)]
    authority: Signer<'info>,
}

#[derive(BorshSerialize, BorshDeserialize)]
pub struct User {
    authority: Pubkey,
}
```

<!-- ![example program: initialization_insecure](https://static.wixstatic.com/media/935a00_ddf79fa835384feeab537745b953b160~mv2.png/v1/fill/w_1480,h_1058,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_ddf79fa835384feeab537745b953b160~mv2.png) -->

The code is directly reading in the account on line 11, then setting the fields. **The program blindly overwrites data whether it is writing for the first time or the second time (or third time).**

Instead, the nomenclature for "initialize" here is "write to the account for the first time".

The "reinitialization attack" here is a different variety from what the Anchor frame is warning about. Specifically, "initialize" can be called several times. Anchor's `init` macro checks that the lamport balance is nonzero and that the program already owns the account, which would prevent multiple calls to `initialize`. The init macro can see the account already has lamports or is owned by the program. However, the code above has no such checks.

It is worth going through their tutorial to see this variety of a reinitialization attack.

Note that this uses an older version of Anchor. The `AccountInfo` is another term for `UncheckedAccount`, so you will need to add a `/// Check:` comment above it.

## Erasing the account discriminator will not make the account reinitializable
Whether or not an account is initialized has nothing to do with the data (or lack thereof) inside of it.

To erase the data in an account without transferring it:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;
use anchor_lang::system_program;

declare_id!("FC467mPCCKXG97ut1WdLLi55vuAcyCW8AD1vid27bZfn");

#[program]
pub mod reinit_attack {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn erase(ctx: Context<Erase>) -> Result<()> {
        ctx.accounts.my_pda.realloc(0, false)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Erase<'info> {
    /// CHECK: We are going to erase the account
    #[account(mut)]
    pub my_pda: UncheckedAccount<'info>,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8, seeds = [], bump)]
    pub my_pda: Account<'info, MyPDA>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyPDA {}
```

It is important that we erase the data using an `UncheckedAccount` as `.realloc(0, false)` is not a method available on a regular `Account`.

This operation will erase the account discriminator, so it will not be readable via `Account` anymore.

**Exercise**: initialize the account, call `erase` then try to initialize the account again. It will fail because even though the account has no data, it is still owned by the program and has a non-zero lamport balance.

## Summary
The `init_if_needed` macro can be convenient to avoid needing two transactions to interact with a new storage account. The Anchor framework blocks it by default to force us to consider the following possible undesirable situations:
- If there is a method to reduce the lamport balance to zero or transfer ownership to the system program, then the account can be re-initialized. This may or may not be a problem depending on the business requirements.
- If the program has both an `init` macro and a `init_if_needed` macro, the developer must ensure that having two codepaths doesn't result in unexpected state.
- Even after the data in an account is completely erased, the account is still initialized.
- If the program has a function that "blindly" writes to an account, then data in that account could get overwritten. This usually requires loading in the account via `AccountInfo` or its alias `UncheckedAccount`.

## Learn more with RareSkills
See our [Solana development course](https://www.rareskills.io/solana-tutorial) for the rest of our Solana tutorials. Thank you for reading!

*Originally Published March, 8, 2024*