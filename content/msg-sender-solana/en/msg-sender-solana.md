# Tx.origin, msg.sender, and onlyOwner in Solana: identifying the caller

![tx.origin msg.sender onlyOwner in Solana](https://static.wixstatic.com/media/935a00_5b75852d244b42ffa7e441959fac0b18~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5b75852d244b42ffa7e441959fac0b18~mv2.jpg)

In Solidity, the `msg.sender` is a global variable that represents the address that called or initiated a function call on a smart contract. The global variable `tx.origin` is the wallet that signed the transaction.

In Solana, there is no equivalent to `msg.sender`.

There is an equivalent to `tx.origin` but you should be aware that Solana transactions can have multiple signers, so we could think of it as having "multiple tx.origins".

To get the "`tx.origin`" address in Solana, you need to set it up by adding Signer account to the function context and pass the caller's account to it when calling the function.

Let's see an example of how we can access the transaction signer's address in Solana:
```rust
use anchor_lang::prelude::*;

declare_id!("Hf96fZsgq9R6Y1AHfyGbhi9EAmaQw2oks8NqakS6XVt1");

#[program]
pub mod day14 {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let the_signer1: &mut Signer = &mut ctx.accounts.signer1;

        // Function logic....

        msg!("The signer1: {:?}", *the_signer1.key);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub signer1: Signer<'info>,
}
```

From the above code snippet, the `Signer<'info>` is used to verify that the `signer1` account in the `Initialize<'info>` account struct has signed the transaction.

In the `initialize` function, the `signer1` account is mutably referenced from the context and assigned to `the_signer1` variable.

Then lastly, we logged the `signer1`'s pubkey (address) using the `msg!` macro and passing in `*the_signer1.key` , which dereferences and access the `key` field or method on the actual value being pointed to by `the_signer1`.

Next is to write a test for the above program:
```typescript
describe("Day14", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Day14 as Program<Day14>;

  it("Is signed by a single signer", async () => {
    // Add your test here.
    const tx = await program.methods.initialize().accounts({
      signer1: program.provider.publicKey
    }).rpc();

    console.log("The signer1: ", program.provider.publicKey.toBase58());
  });
});
```

In the test, we passed our wallet account as signer to the `signer1` account, then called the initialize function. Following that, we logged the wallet account on the console to verify its consistency with the one in our program.

**Exercise:** What did you notice from the outputs in **shell_1** (commands terminal) and **shell_3** (logs terminal) after running the test?

## Multiple signers
In Solana, we can also have more than one signer sign a transaction, you can think of this as batching up a bunch of signatures and sending it in one transaction. One use-case is doing a multisig transaction in one transaction.

To do that, we just add more Signer structs to the account struct in our program, then ensure the necessary accounts are passed when calling the function:
```rust
use anchor_lang::prelude::*;

declare_id!("Hf96fZsgq9R6Y1AHfyGbhi9EAmaQw2oks8NqakS6XVt1");

#[program]
pub mod day14 {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let the_signer1: &mut Signer = &mut ctx.accounts.signer1;
        let the_signer2: &mut Signer = &mut ctx.accounts.signer2;

        msg!("The signer1: {:?}", *the_signer1.key);
        msg!("The signer2: {:?}", *the_signer2.key);

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    pub signer1: Signer<'info>,
    pub signer2: Signer<'info>,
}
```
The above example is somewhat the same as the single signer example, with one notable difference. In this case, we added another Signer account (`signer2`) to the `Initialize` struct and also logged both signers pubkey in the **initialize** function.

Calling the **initialize** function with multiple signers is different, compared to a single signer. The test below shows how to invoke a function with multiple signers:
```typescript
describe("Day14", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Day14 as Program<Day14>;

  // generate a signer to call our function
  let myKeypair = anchor.web3.Keypair.generate();

  it("Is signed by multiple signers", async () => {
    // Add your test here.
    const tx = await program.methods
      .initialize()
      .accounts({
        signer1: program.provider.publicKey,
        signer2: myKeypair.publicKey,
      })
      .signers([myKeypair])
      .rpc();

    console.log("The signer1: ", program.provider.publicKey.toBase58());
    console.log("The signer2: ", myKeypair.publicKey.toBase58());
  });
});
```
So what is different about the above test? First is the `signers()` method, which takes in an array of signers which signs a transaction as an argument. But we only have one signer in the array, instead of two. Anchor automatically passes the wallet account in the provider as a signer, so we don't need to add it to the signers array again.

### Generating random addresses to test with
The second change is the `myKeypair` variable, which stores the Keypair (*A publickey and corresponding private key for accessing an account*) that is randomly generated by the `anchor.web3` module. In the test, we assigned the Keypair's (which is stored in the `myKeypair` variable) publickey to the `signer2` account, that is why it is passed as argument in the `.signers([myKeypair])` method.

Run the test multiple times, you will notice that `signer1` pubkey does not change but `signer2` pubkey changes. This is because the wallet account assigned to the `signer1` account (in the test) is from the provider, which is also the Solana wallet account in your local machine and the account assigned to `signer2` is randomly generated each time you run `anchor test —skip-local-validator`.

**Exercise:** Create another function (you can call it whatever) that requires three signers (the provider wallet account and two randomly generated accounts) and write a test for it.

## onlyOwner
This is a common pattern used in Solidity to restrict a function's access to only the owner of the contract. Using `#[access_control]` attribute from Anchor, we can also implement the only owner pattern, that is, restrict a function's access in our Solana program to a PubKey (owner's address).

Here's an example of how to implement "onlyOwner" functionality in Solana:
```rust
use anchor_lang::prelude::*;

declare_id!("Hf96fZsgq9R6Y1AHfyGbhi9EAmaQw2oks8NqakS6XVt1");

// NOTE: Replace with your wallet's public key
const OWNER: &str = "8os8PKYmeVjU1mmwHZZNTEv5hpBXi5VvEKGzykduZAik";

#[program]
pub mod day14 {
    use super::*;

    #[access_control(check(&ctx))]
    pub fn initialize(ctx: Context<OnlyOwner>) -> Result<()> {
        // Function logic...

        msg!("Holla, I'm the owner.");
        Ok(())
    }
}

fn check(ctx: &Context<OnlyOwner>) -> Result<()> {
    // Check if signer === owner
    require_keys_eq!(
        ctx.accounts.signer_account.key(),
        OWNER.parse::<Pubkey>().unwrap(),
        OnlyOwnerError::NotOwner
    );

    Ok(())
}

#[derive(Accounts)]
pub struct OnlyOwner<'info> {
    signer_account: Signer<'info>,
}

// An enum for custom error codes
#[error_code]
pub enum OnlyOwnerError {
    #[msg("Only owner can call this function!")]
    NotOwner,
}
```
In the context of the code above, the `OWNER` variable stores the pubkey (address) associated with my local Solana wallet. Be sure to replace the OWNER variable with your wallet's pubkey before testing. You can easily retrieve your pubkey by running the `solana address` command.


The `#[access_control]` attribute executes the given access control method before running the main instruction. When the initialize function is called, the access control method (`check`) is executed prior to the initialize function. The `check` method accepts a referenced context as argument, then it checks if the signer of the transaction equals the value of the `OWNER` variable. The `require_keys_eq!` macro ensures two pubkeys values are equal, if true, it executes the initialize function, else, it reverts with the `NotOwner` custom error.

### Testing the onlyOwner functionality — happy case
In the test below, we are calling the initialize function and signing the transaction using the owner's keypair:
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Day14 } from "../target/types/day14";

describe("day14", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Day14 as Program<Day14>;

  it("Is called by the owner", async () => {
    // Add your test here.
    const tx = await program.methods
      .initialize()
      .accounts({
        signerAccount: program.provider.publicKey,
      })
      .rpc();

    console.log("Transaction hash:", tx);
  });
});
```
We called the initialize function and passed the wallet account (*local Solana wallet account*) in the provider to the `signerAccount` which has the `Signer<'info>` struct, to validate that the wallet account actually signed the transaction. Also remember that Anchor secretly signs any transaction using the wallet account in the provider.

Run test `anchor test --skip-local-validator` , if everything was done correctly, the test should pass:
![Anchor test passing](https://static.wixstatic.com/media/935a00_a6f122cf3fdf49ac98dbb04d90495aa4~mv2.png/v1/fill/w_1202,h_304,al_c,q_90,enc_auto/935a00_a6f122cf3fdf49ac98dbb04d90495aa4~mv2.png)

### Testing if the signer is not the owner — attack case
Using a different keypair that is not the owner to call the initialize function and sign the transaction will throw an error since the function call is restricted to only the owner:
```typescript
describe("day14", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Day14 as Program<Day14>;

  let Keypair = anchor.web3.Keypair.generate();

  it("Is NOT called by the owner", async () => {
    // Add your test here.
    const tx = await program.methods
      .initialize()
      .accounts({
        signerAccount: Keypair.publicKey,
      })
      .signers([Keypair])
      .rpc();

    console.log("Transaction hash:", tx);
  });
});
```
Here we generated a random keypair and used it to sign the transaction. Let's run test again:
![anchor test fail due to wrong signer](https://static.wixstatic.com/media/935a00_b2f1fa2f9e5049998817b69db9025b42~mv2.png/v1/fill/w_1210,h_342,al_c,q_90,enc_auto/935a00_b2f1fa2f9e5049998817b69db9025b42~mv2.png)
As expected, we got an error, since the signer's pubkey is not equal to the owner's pubkey.

### Modify the owner
To change the owner in a program, the pubkey assigned to the owner needs to be stored on-chain. However, discussions about "storage" in Solana will be covered in a future tutorial.

The owner can just redeploy the bytecode.

**Exercise**: Upgrade a program like the one above to have a new owner.

## Learn more with RareSkills
This tutorial is chapter 14 in our [Solana course](https://www.rareskills.io/solana-tutorial).

*Originally Published February, 21, 2024*