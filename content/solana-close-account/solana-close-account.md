# Deleting and Closing Accounts and Programs in Solana

![Hero image showing Close accounts and programs](https://static.wixstatic.com/media/935a00_74aadefdf66141ac8156b6fb8a78cbfd~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_74aadefdf66141ac8156b6fb8a78cbfd~mv2.jpg)

In the Anchor framework for Solana, `close` is the opposite of `init` ([initializing an account in Anchor](https://www.rareskills.io/post/solana-initialize-account)) — it reduces the lamport balance to zero, sending the lamports to a target address, and changes the owner of the account to be the system program.

Here is an example of using the `close` instruction in Rust:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("8gaSDFr5cVy2BkLrWfSX9MCtPX9N4gmXDvTVm7RS6DYK");

#[program]
pub mod close_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn delete(ctx: Context<Delete>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = size_of::<ThePda>() + 8, seeds = [], bump)]
    pub the_pda: Account<'info, ThePda>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Delete<'info> {
    #[account(mut, close = signer, )]
    pub the_pda: Account<'info, ThePda>,

    #[account(mut)]
    pub signer: Signer<'info>,
}

#[account]
pub struct ThePda {
    pub x: u32,
}
```

### Solana returns rent for closing accounts
The `close = signer` macro specifies that the signer in the transaction will receive the rent that was set aside to pay for storage (though another address could be specified of course). This is similar to how selfdestruct in Ethereum (prior to the Dencun upgrade) refunded users for clearing space. The amount of SOL that can be earned from closing an account is proportional to how large the account was.

Here is the Typescript to call `initialize` followed by `delete`:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { CloseProgram } from "../target/types/close_program";
import { assert } from "chai";

describe("close_program", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.CloseProgram as Program<CloseProgram>;

  it("Is initialized!", async () => {
    let [thePda, _bump] = anchor.web3.PublicKey.findProgramAddressSync([], program.programId);
    await program.methods.initialize().accounts({thePda: thePda}).rpc();
    await program.methods.delete().accounts({thePda: thePda}).rpc();

    let account = await program.account.thePda.fetchNullable(thePda);
    console.log(account)
  });
});
```

The `close = signer` instruction says to send the rent lamports to the signer, but you can specify whichever address you prefer.

**The above construction allows anyone to close the account**, you probably want to add some kind of access control in a real application!

## Accounts can be initialized after being closed
If you call `initialize` after closing an account, it will be initialized again. Of course, the rent, which was redeemed earlier, must be paid again.

**Exercise**: add another call to `initialize` in the unit test to see it pass. Note that the account is no longer null at the end of the test.

## What does close do under the hood?
If we look at the [source code for the close command](https://github.com/coral-xyz/anchor/blob/v0.29.0/lang/src/common.rs) in Anchor, we can see it doing the operations we described above:

![Close : lamports](https://static.wixstatic.com/media/935a00_dfd66357bad44b758fce6240bebae673~mv2.png/v1/fill/w_1480,h_708,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_dfd66357bad44b758fce6240bebae673~mv2.png)

## Many Anchorlang examples are outdated
In version 0.25 of Anchor, the close sequence was different.

Similar to the current implementation, it would first send all the lamports to the destination address.

However, instead of erasing the data and transferring it to the system program, `close` would write a special 8 byte sequence called the `CLOSE_ACCOUNT_DISCRIMINATOR`. ([original code](https://github.com/coral-xyz/anchor/blob/v0.25.0/lang/src/lib.rs#L273)):

```bash
/// The discriminator anchor uses to mark an account as closed.
pub const CLOSED_ACCOUNT_DISCRIMINATOR: [u8; 8] = [255, 255, 255, 255, 255, 255, 255, 255];
```

<!-- ![pub const CLOSED_ACCOUNT_DISCRIMINATOR](https://static.wixstatic.com/media/935a00_24b182dead824479901e064b4ae16dda~mv2.png/v1/fill/w_1480,h_84,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_24b182dead824479901e064b4ae16dda~mv2.png) -->

Eventually, the runtime would erase the account because it had zero lamports.

### What is the account discriminator in Anchor?
When Anchor initializes an account, it computes the discriminator and stores that in the first 8 bytes of the account. The account discriminator is the first 8 bytes of the SHA256 of the Rust identifier of the struct.

When a user asks the program to load an account via `pub the_pda: Account<'info, ThePda>`, the program will compute the first 8 bytes of the SHA256 of the `ThePda` identifier. Then it will load `ThePda` data and compare the discriminator stored there to the one it computed. If they do not match, then Anchor will not deserialize the account.

The intent here is to prevent an attacker from crafting a malicious account which will deserialize into unexpected results when parsed "through the wrong struct."

### Why Anchor used to set the account discriminator to `[255, ..., 255]`
By setting the account discriminator to all ones, then Anchor will always reject deserializing the account because it will not match any of the account discriminators.

The reason for writing the account discriminator as all ones was to prevent an attacker from sending SOL directly to the account before the runtime erased it. Under this circumstance, the program "thought" it closed the program, but the attacker "revived" it. If the old account discriminator is still there, then the data which was thought to be deleted will be read back in.

### Why setting the account discriminator to `[255, …, 255]` is no longer needed
By instead changing ownership to the system program, reviving the account won't result in the program suddenly "owning" the account again, the system program owns the revived account and the attacker wasted SOL.

To change ownership back to the program, it needs to be explicitly initialized again, it cannot be revived via a side-channel like sending SOL to prevent the runtime from erasing it.

## Closing a program via CLI

To close a program, as opposed to an account owned by it, we can use the command-line:

```bash
solana program close <address> --bypass warning
```

The warning is that once a program is closed, a program with the same address cannot be recreated. Here is a sequence of shell commands illustrating closing an account:

![solona program close](https://static.wixstatic.com/media/935a00_6656a12dd8ab418eb568038dc955fbeb~mv2.png/v1/fill/w_1480,h_470,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_6656a12dd8ab418eb568038dc955fbeb~mv2.png)

Here is the sequence of commands in the screenshot above:
1. First we deploy the program
2. We close the program without the `--bypass-warning` flag and the tool gives us a warning that the program cannot be deployed again
3. We close the program with the flag, the program is closed, and we receive 2.918 SOL as a refund for closing the account
4. We try to deploy again and fail because a closed program cannot be redeployed

## Learn more with RareSkills
To continue learning Solana development, please see our [Solana course](https://www.rareskills.io/solana-tutorial). For other blockchain topics, see our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps).

*Originally Published March, 12, 2024*