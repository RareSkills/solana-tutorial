# Reading an account balance in Anchor: address(account).balance in Solana

![Solona and Anchor get account balance](https://static.wixstatic.com/media/935a00_8742c65f9a7a48e3b6db4aa916303cd0~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_8742c65f9a7a48e3b6db4aa916303cd0~mv2.jpg)

## Reading an account balance in Anchor Rust
To read the Solana balance of an address inside a Solana program, use the following code:

```rust
use anchor_lang::prelude::*;

declare_id!("Gnf6u7S7fGJbqEGH9PuDE5Prq6f6ZrDxHY3jNJ4SYySQ");

#[program]
pub mod balance {
    use super::*;

    pub fn read_balance(ctx: Context<ReadBalance>) -> Result<()> {
        let balance = ctx.accounts.acct.to_account_info().lamports();

        msg!("balance in Lamports is {}", balance);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct ReadBalance<'info> {
    /// CHECK: although we read this account's balance, we don't do anything with the information
    pub acct: UncheckedAccount<'info>,
}
```
And the following is the web3 js code to trigger it:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Balance } from "../target/types/balance";

describe("balance", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Balance as Program<Balance>;

  // the following is the Solana wallet we are using
  let pubkey = new anchor.web3.PublicKey("5jmigjgt77kAfKsHri3MHpMMFPo6UuiAMF19VdDfrrTj");


  it("Tests the balance", async () => {
    const tx = await program.methods.readBalance().accounts({ acct: pubkey }).rpc();
  });
});
```

Some items in this example differ from earlier tutorials, particularly the use of `UncheckedAccount`.

## What is `UncheckedAccount` in Solana Anchor?
The `UncheckedAccount` type tells to Anchor to not check if the account being read is owned by the program.

Note that the account we passed through the `Context` struct is not an account that this program initialized, hence the program does not own it.

When Anchor reads an account of type `Account` in the `#[derive(Accounts)]` it will check (behind the scenes) if that account is owned by that program. If not, the execution will halt.

This serves as an important safety check.

**If a malicious user crafts an account the program did not create and then passes it to the Solana program, and the Solana program blindly trusts the data in the account, critical errors may occur.**

For example, if the program is a bank, and the account is storing how much balance a user has, then a hacker could supply a *different* account with an artificially higher balance than they actually have.

To pull off this hack, however, the user would have to create the false account in a separate transaction, then pass it to the Solana program. However, the Anchor framework checks behind the scenes to see if the account is not owned by the program, and rejects reading the account.

`UncheckedAccount` bypasses this safety check.

**Important**: `AccountInfo` and `UncheckedAccount` are aliases for each other and `AccountInfo` has the same security considerations.

In our case, we are passing in accounts that are certainly not owned by the program — we want to check the balance of an *arbitrary account*. Therefore, we must be certain that no critical logic can be tampered with with this safety check removed.

In our case, we are just logging the balance to the console, but most real-world use cases will have more complex logic.

### What is `/// CHECK:`?
Because of the danger of using `UncheckedAccount`, Anchor forces you to include this comment to encourage you not to ignore the safety considerations.

**Exercise**: remove the `/// Check:` comment and run `anchor build` you should see the build halt and ask you to add the comment back with an explanation for why an Unchecked Account is safe. That is, reading in an untrusted account could be dangerous, Anchor wants to make sure you are not doing anything critical with the data in the account.

## Why is there no `#[account]` struct in the program?
The `#[account]` struct tells Anchor how to deserialize an account holding data. For example, an account struct that looks like the following will inform Anchor that it should deserialize the data stored in the account into a single `u64`:

```rust
#[account]
pub struct Counter {
    counter: u64
}
```

In our case however, we are not reading the data from the account — we are only reading the balance. This is similar to how this similar to how we can read the balance of an Ethereum address but not read any of it's code. Since we do *not* want to deserialize the data, we don't supply an `#[account]` struct.

## Not all the SOL in an account is spendable
Recall from our discussion of [Solana account rent](https://www.rareskills.io/post/solana-account-rent) that the account must maintain a certain balance of SOL to be "rent exempt" or the runtime will delete the account. Just because the account has "1 SOL" in it does not necessarily mean the account can spend the entire 1 SOL.

For example, if you are building a staking or bank application where a user's deposited SOL is kept in separate accounts, it is not accurate to simply measure the SOL balance of those accounts as the rent will be included in the balance.

## Learn more with RareSkills
See our [Solana developer course](https://www.rareskills.io/solana-tutorial) for more Solana materials.

*Originally Published February, 29, 2024*