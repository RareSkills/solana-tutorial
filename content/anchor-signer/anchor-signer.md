# Modifying accounts using different signers

![Hero image showing Anchor Signer: Modifying accounts with different signers](https://static.wixstatic.com/media/935a00_8a5622df6d344ca3bd3d548454a703fe~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_8a5622df6d344ca3bd3d548454a703fe~mv2.jpg)

In our [Solana tutorials](https://www.rareskills.io/solana-tutorial) thus far, we've only had one account initialize and write to the account.

In practice, this is very restrictive. For example, if user Alice is transferring points to Bob, Alice must be able to write to an account initialized by user Bob.

In this tutorial we will demonstrate initializing an account with one wallet and updating it with another.

## The initialization step
The Rust code we've been using to initialize accounts doesn't change:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("61As9Y8pREgvFZzps6rpFai8UkageeHT6kW1dnGRiefb");

#[program]
pub mod other_write {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init,
              payer = signer,
              space=size_of::<MyStorage>() + 8,
              seeds = [],
              bump)]
    pub my_storage: Account<'info, MyStorage>,

    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

### Doing the initialization transaction with an alternate wallet
However, there is an important change in the client code:
- For testing purposes, we create a new wallet called `newKeypair`. This is different from the one Anchor provides by default.
- We airdrop that new wallet 1 SOL so it can pay for transactions.
- Pay attention to the comment `// THIS MUST BE EXPLICITLY SPECIFIED`. We are passing the publicKey of that wallet for the `Signer` field. When we use the default signer built into Anchor, Anchor passes this in the background for us. However, when we use a different wallet, we need to provide this explictily.
- We set the signer to be `newKeypair` with the `.signers([newKeypair])` configuration.

We will explain after this code snippet why we are (seemingly) specifying the signer twice:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { OtherWrite } from "../target/types/other_write";

// this airdrops sol to an address
async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}

describe("other_write", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.OtherWrite as Program<OtherWrite>;

  it("Is initialized!", async () => {
    const newKeypair = anchor.web3.Keypair.generate();
    await airdropSol(newKeypair.publicKey, 1e9); // 1 SOL

    let seeds = [];
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);
    
    await program.methods.initialize().accounts({
      myStorage: myStorage,
      signer: newKeypair.publicKey // ** THIS MUST BE EXPLICITLY SPECIFIED **
    }).signers([newKeypair]).rpc();
  });
});
```
It is not required that the key `signer` be called `signer`.

**Exercise**:  In the Rust code, change `payer = signer` to `payer = fren` and `pub signer: Signer<'info>`, to `pub fren: Signer<'info>`, and change `signer: newKeypair.publicKey` to `fren: newKeypair.publicKey` in the test. The initialization should succeed and the test should pass.

## Why does Anchor require specifying the Signer and the publicKey?
At first it might seem redundant that we are specifying the signer twice, but let's take a closer look:

![A public key is passed here](https://static.wixstatic.com/media/935a00_e6738de42e754598bdc0b91d6b161675~mv2.png/v1/fill/w_1480,h_1372,al_c,q_95,usm_0.66_1.00_0.01,enc_auto/935a00_e6738de42e754598bdc0b91d6b161675~mv2.png)

In the <span style="color:red">red box</span>, we see the `fren` field specified to be a Signer account. **The `Signer` type means Anchor will look at the signature of the transaction and make sure the signature matches the address passed here.**

We will see later how we can use this to validate the Signer is authorized to conduct certain a transaction.

Anchor has been doing this the whole time behind the scenes, but since we passed in a `Signer` other than the one Anchor uses by default, we have to be explicit about what account the `Signer` is.

## Error: unknown signer in Solana Anchor
The `unknown signer` error occurs when the signer of the transaction does not match the public key passed to `Signer`.

Suppose we modify the test to remove the `.signers([newKeypair])` spec. Anchor will use the default signer instead, and the default signer will not match the `publicKey` of our `newKeypair` wallet:

![Removing .signers([newKeypair])](https://static.wixstatic.com/media/935a00_2b7cffddcb2a490aaea4396ee71a47a1~mv2.png/v1/fill/w_1480,h_334,al_c,lg_1,q_90,enc_auto/935a00_2b7cffddcb2a490aaea4396ee71a47a1~mv2.png)

We will get the following error:

![Error: Signature verification fail](https://static.wixstatic.com/media/935a00_e65c13cd81be466dbb8abed42a18c3f4~mv2.png/v1/fill/w_1480,h_294,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_e65c13cd81be466dbb8abed42a18c3f4~mv2.png)

Similarly, if we don't pass in the `publicKey` explicitly, Anchor will silently use the default signer:

![Dont pass publicKey](https://static.wixstatic.com/media/935a00_3d3ff1498fb74e98a9d153733f621ade~mv2.png/v1/fill/w_1416,h_341,al_c,lg_1,q_90,enc_auto/935a00_3d3ff1498fb74e98a9d153733f621ade~mv2.png)

And we will get the following `Error: unknown signer`:

![Error: unknown signer](https://static.wixstatic.com/media/935a00_6cffaab9b57f4f40ad296c048769d223~mv2.png/v1/fill/w_1480,h_284,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_6cffaab9b57f4f40ad296c048769d223~mv2.png)

Somewhat misleadingly, Anchor isn't saying the signer is unknown because it wasn't specified per se. Anchor is able to figure out that if no signer is specified, then it will use the default signer. If we remove both the `.signers([newKeypair])` code *and* the `fren: newKeypair.publicKey` code, then Anchor will use the default signer for both the public key to check against, and the signature of the signer to verify it matches the public key.

The following code will result in a successful initialization because both the `Signer` public key and the account that signs the transaction are the Anchor default signer.

```typescript
  await program.methods.initialize().accounts({
    myStorage: myStorage
  }).rpc();
});
```

<!-- ![Successful initialize](https://static.wixstatic.com/media/935a00_f4422a244375428b8b18b9990d7aa1d3~mv2.png/v1/fill/w_866,h_144,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f4422a244375428b8b18b9990d7aa1d3~mv2.png) -->

![other_write initialized](https://static.wixstatic.com/media/935a00_8c4141658d744bfe980e6063b598eb6d~mv2.png/v1/fill/w_864,h_202,al_c,lg_1,q_85,enc_auto/935a00_8c4141658d744bfe980e6063b598eb6d~mv2.png)

## Bob can write to an account Alice initialized
Below we show an Anchor program with functions to initialize an account and write to it.

This will be familiar from our Solana counter program tutorial, but pay attention to the small addition marked by the `// THIS FIELD MUST BE INCLUDED` comment near the bottom:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("61As9Y8pREgvFZzps6rpFai8UkageeHT6kW1dnGRiefb");

#[program]
pub mod other_write {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn update_value(ctx: Context<UpdateValue>, new_value: u64) -> Result<()> {
        ctx.accounts.my_storage.x = new_value;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init,
              payer = fren,
              space=size_of::<MyStorage>() + 8,
              seeds = [],
              bump)]
    pub my_storage: Account<'info, MyStorage>,

    #[account(mut)]
    pub fren: Signer<'info>, // A public key is passed here

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateValue<'info> {
    #[account(mut, seeds = [], bump)]
    pub my_storage: Account<'info, MyStorage>,

    // THIS FIELD MUST BE INCLUDED
    #[account(mut)]
    pub fren: Signer<'info>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

The following client code will create a wallet for Alice and Bob and airdrop them 1 SOL each. Alice will initialize the account `MyStorage`, and Bob will write to it:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { OtherWrite } from "../target/types/other_write";

// this airdrops sol to an address
async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}

describe("other_write", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.OtherWrite as Program<OtherWrite>;

  it("Is initialized!", async () => {
    const alice = anchor.web3.Keypair.generate();
    const bob = anchor.web3.Keypair.generate();

    const airdrop_alice_tx = await anchor.getProvider().connection.requestAirdrop(alice.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_tx);

    const airdrop_alice_bob = await anchor.getProvider().connection.requestAirdrop(bob.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_bob);

    let seeds = [];
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);
    
    // ALICE INITIALIZE ACCOUNT
    await program.methods.initialize().accounts({
      myStorage: myStorage,
      fren: alice.publicKey
    }).signers([alice]).rpc();

    // BOB WRITE TO ACCOUNT
    await program.methods.updateValue(new anchor.BN(3)).accounts({
      myStorage: myStorage,
      fren: bob.publicKey
    }).signers([bob]).rpc();

    let value = await program.account.myStorage.fetch(myStorage);
    console.log(`value stored is ${value.x}`);
  });
});
```

## Restricting writes to Solana accounts
In real applications, we don't want Bob writing arbitrary data to arbitrary accounts. Let's create a basic example where users can initialize an account with 10 points and transfer those points to another account. (There is an obvious problem that a hacker can create as many accounts as they want using separate wallets, but that is outside of the scope of our example).

## Building a proto-ERC20 program
Alice should be able to modify both her account and Bob's account. That is, she should be able to deduct her points and credit Bob's points. She should not be able to deduct Bob's points — only Bob should be able to do that.

By convention, we call an address that can make privileged  changes to an account an "authority" in Solana. It is a common pattern to store the "authority" field in the account struct to signify that only that account can conduct sensitive operations on that account (such as deducting points in our example).

This is somewhat analogous to the [onlyOwner pattern in Solidity](https://www.rareskills.io/post/anchor-signer), except that instead of applying to the entire contract, it applies only to a single account:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("HFmGQX4wPgPYVMFe4WrBi925NKvGySrEG2LGyRXsXJ4Z");

const STARTING_POINTS: u32 = 10;

#[program]
pub mod points {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.player.points = STARTING_POINTS;
        ctx.accounts.player.authority = ctx.accounts.signer.key();
        Ok(())
    }

    pub fn transfer_points(ctx: Context<TransferPoints>,
                           amount: u32) -> Result<()> {
        require!(ctx.accounts.from.authority == ctx.accounts.signer.key(), Errors::SignerIsNotAuthority);
        require!(ctx.accounts.from.points >= amount, Errors::InsufficientPoints);
        
        ctx.accounts.from.points -= amount;
        ctx.accounts.to.points += amount;
        Ok(())
    }
}

#[error_code]
pub enum Errors {
    #[msg("SignerIsNotAuthority")]
    SignerIsNotAuthority,
    #[msg("InsufficientPoints")]
    InsufficientPoints
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init,
              payer = signer,
              space = size_of::<Player>() + 8,
              seeds = [&(signer.as_ref().key().to_bytes())],
              bump)]
    player: Account<'info, Player>,
    #[account(mut)]
    signer: Signer<'info>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct TransferPoints<'info> {
    #[account(mut)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    #[account(mut)]
    signer: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}
```
Note that we use the address of the signer (`&(signer.as_ref().key().to_bytes())`) to derive the address of the account where their points are stored. This behaves like a Solidity [mapping in Solana](https://www.rareskills.io/post/solana-solidity-mapping), where the [Solana "msg.sender / tx.origin"](https://www.rareskills.io/post/msg-sender-solana) is the key.

In the `initialize` function, the program sets the initial points to `10` and the authority to the `signer`. The user does not have control over these initial values.

The `transfer_points` function uses [Solana Anchor require macros and error code macros](https://www.rareskills.io/post/solana-require-macro) to ensure that 1) the Signer of the transaction is the authority of the account whose balance is getting deducted; and 2) the account has enough points balance to transfer.

The test codebase should be straightforward to understand. Alice and Bob initialize their accounts, then Alice transfer 5 points to Bob:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Points } from "../target/types/points";

// this airdrops sol to an address
async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount);
  await confirmTransaction(airdropTx);
}

async function confirmTransaction(tx) {
  const latestBlockHash = await anchor.getProvider().connection.getLatestBlockhash();
  await anchor.getProvider().connection.confirmTransaction({
    blockhash: latestBlockHash.blockhash,
    lastValidBlockHeight: latestBlockHash.lastValidBlockHeight,
    signature: tx,
  });
}

describe("points", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.Points as Program<Points>;


  it("Alice transfers points to Bob", async () => {
    const alice = anchor.web3.Keypair.generate();
    const bob = anchor.web3.Keypair.generate();

    const airdrop_alice_tx = await anchor.getProvider().connection.requestAirdrop(alice.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_tx);

    const airdrop_alice_bob = await anchor.getProvider().connection.requestAirdrop(bob.publicKey, 1 * anchor.web3.LAMPORTS_PER_SOL);
    await confirmTransaction(airdrop_alice_bob);

    let seeds_alice = [alice.publicKey.toBytes()];
    const [playerAlice, _bumpA] = anchor.web3.PublicKey.findProgramAddressSync(seeds_alice, program.programId);

    let seeds_bob = [bob.publicKey.toBytes()];
    const [playerBob, _bumpB] = anchor.web3.PublicKey.findProgramAddressSync(seeds_bob, program.programId);

    // Alice and Bob initialize their accounts
    await program.methods.initialize().accounts({
      player: playerAlice,
      signer: alice.publicKey,
    }).signers([alice]).rpc();

    await program.methods.initialize().accounts({
      player: playerBob,
      signer: bob.publicKey,
    }).signers([bob]).rpc();

    // Alice transfers 5 points to Bob. Note that this is a u32
    // so we don't need a BigNum
    await program.methods.transferPoints(5).accounts({
      from: playerAlice,
      to: playerBob,
      signer: alice.publicKey,
    }).signers([alice]).rpc();

    console.log(`Alice has ${(await program.account.player.fetch(playerAlice)).points} points`);
    console.log(`Bob has ${(await program.account.player.fetch(playerBob)).points} points`)
  });
});
```
**Exercise**: Create a keypair `mallory` and try to get `mallory` to steal points from Alice or Bob by using `mallory` as the signer in `.signers([mallory])`. Your attack should fail, but you should try anyway.

## Using Anchor Constraints to replace require! macros
An alternative to writing `require!(ctx.accounts.from.authority == ctx.accounts.signer.key(), Errors::SignerIsNotAuthority);` is to use an Anchor constraint. The Anchor account docs give us a list of constraints available to us.

## Anchor `has_one` constraint
The `has_one` constraint assumes that there is "shared key" between `#[derive(Accounts)]` and `#[account]` and checks that both of those keys have the same value. The best way to demonstrate this is with a picture:

![pup struct TransferPoints](https://static.wixstatic.com/media/935a00_bfee9678192b41db9831e3adc6016eb2~mv2.png/v1/fill/w_664,h_612,al_c,q_90,enc_auto/935a00_bfee9678192b41db9831e3adc6016eb2~mv2.png)

Behind the scenes, Anchor will block the transaction if the `authority` account passed as part of the transaction (as the `Signer`) is not equal to the `authority` stored in the account.

In our implementation above, we used the key `authority` in the account and `signer` in the `#[derive(Accounts)]`. This mismatch of key names will prevent this macro from working, so the code above changes the key `signer` to `authority`. Authority is not a special keyword, merely a convention. You could, as an exercise, change all instances of `authority` to fren and the code will work the same.

## Anchor `constraint` constraint
We can also replace the macro `require!(ctx.accounts.from.points >= amount, Errors::InsufficientPoints);` with an Anchor constraint.

The `constraint` macro allows us to place arbitrary constraints on accounts passed to the transactions and data in the account. In our case, we want to make sure the sender has enough points:

```rust
#[derive(Accounts)]
#[instruction(amount: u32)] // amount must be passed as an instruction
pub struct TransferPoints<'info> {
    #[account(mut,
              has_one = authority,
              constraint = from.points >= amount)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    authority: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}
```

The macro is smart enough to recognize that `from` is based on the account passed into the `from` key, and that account has a `points` field. The `amount` from the `transfer_points` function argument must be passed via the `instruction` macro so the `constraint` macro can compare `amount` to the point balance in the account.

## Adding custom error messages to Anchor constraints
We can improve the readability of error messages when constraints are violated by adding custom errors, the same custom errors we passed to the `require!` macros using the `@` notation:

```rust
#[derive(Accounts)]
#[instruction(amount: u32)]
pub struct TransferPoints<'info> {
    #[account(mut,
              has_one = authority @ Errors::SignerIsNotAuthority,
              constraint = from.points >= amount @ Errors::InsufficientPoints)]
    from: Account<'info, Player>,
    #[account(mut)]
    to: Account<'info, Player>,
    authority: Signer<'info>,
}

#[account]
pub struct Player {
    points: u32,
    authority: Pubkey
}
```
The `Errors` enum was defined in the Rust code earlier that used them in the `require!` macros.

**Exercise**: modify the tests to violate the `has_one` and `constraint` macro and observe the error messages.

## Learn more Solana with RareSkills
Our [Solana tutorials](https://www.rareskills.io/solana-tutorial)covers how to learn Solana as an Ethereum or EVM developer.

*Originally Published March, 5, 2024*