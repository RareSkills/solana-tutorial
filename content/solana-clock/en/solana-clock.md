# The Solana clock and other "block" variables

![solana clock](https://static.wixstatic.com/media/935a00_55b04d2394f04f7781fdee936109b747~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_55b04d2394f04f7781fdee936109b747~mv2.jpg)

Today we will cover the analogs of all the block variables from Solidity. Not all of them have 1-1 analogs. In Solidity, we have the following commonly used block variables:
+ block.timestamp
+ block.number
+ blockhash()

And the lesser known ones:
+ block.coinbase
+ block.basefee
+ block.chainid
+ block.difficulty / block.prevrandao

We assume you already know what they do, but they are explained in the [Solidity global variables doc](https://docs.soliditylang.org/en/latest/units-and-global-variables.html) if you need a refresher.

## block.timestamp in Solana

By utilizing the `unix_timestamp` field within the [Clock sysvar](https://docs.solanalabs.com/runtime/sysvars), we can access the block timestamp Solana.

**First we initialize a new Anchor project:**

```bash
anchor init sysvar
```

**Replace the initialize function with this:**

```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let clock: Clock = Clock::get()?;
    msg!(
        "Block timestamp: {}",
        // Get block.timestamp
        clock.unix_timestamp,
    );
    Ok(())
}
```

Anchor's prelude module contains the Clock struct, which is automatically imported by default:
```rust
use anchor_lang::prelude::*;
```
Somewhat confusingly, the type returned by `unix_timestamp` is an `i64`, not a `u64` meaning it supports negative numbers even though time cannot be negative. Time deltas however can be negative.

### Getting the day of the week
Now let's create a program that tells us the current day of the week using the `unix_timestamp` from the Clock `sysvar`.

The [chrono](https://docs.rs/chrono/latest/chrono/) crate provides functionality for operations on dates and times in Rust.

Add the `chrono` crate as a dependency in our Cargo.toml file in the program directory `./sysvar/Cargo.toml`:

```rust
[dependencies]
chrono = "0.4.31"
```

**Import the `chrono` crate inside the `sysvar` module:**
```rust
// ...other code

#[program]
pub mod sysvar {
    use super::*;
    use chrono::*;  // new line here

    // ...
}
```
**Now, we add this function below to our program:**
```rust
pub fn get_day_of_the_week(
    _ctx: Context<Initialize>) -> Result<()> {

    let clock = Clock::get()?;
    let time_stamp = clock.unix_timestamp; // current timestamp

    let date_time = chrono::NaiveDateTime::from_timestamp_opt(time_stamp, 0).unwrap();
    let day_of_the_week = date_time.weekday();

    msg!("Week day is: {}", day_of_the_week);

    Ok(())
}
```
We pass the current unix timestamp we get from the Clock `sysvar` as an argument to the `from_timestamp_opt` function which returns a `NaiveDateTime` struct containing a date and time. Then we call the weekday method to get the current weekday based on the timestamp we passed.

**And update our test:**
```javascript
it("Get day of the week", async () => {
    const tx = await program.methods.getDayOfTheWeek().rpc();
    console.log("Your transaction signature", tx);
});
```

**Run the test again and get the following logs:**

```bash
Transaction executed in slot 36:
  Signature: 5HVAjmo85Yi3yeQX5t6fNorU1da4H1zvgcJN7BaiPGnRwQhjbKd5YHsVE8bppU9Bg2toF4iVBvhbwkAtMo4NJm7V
  Status: Ok
  Log Messages:
    Program H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy invoke [1]
    Program log: Instruction: GetDayOfTheWeek
    Program log: Week day is: Wed
    Program H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy consumed 1597 of 200000 compute units
```

<!-- ![test output](https://static.wixstatic.com/media/935a00_85c168cb57ae40b592c769bbebf27b2c~mv2.png/v1/fill/w_1480,h_232,al_c,lg_1,q_85,enc_auto/935a00_85c168cb57ae40b592c769bbebf27b2c~mv2.png) -->
Notice the "`Week day is: Wed`" log.

## block.number in Solana
Solana has a notion of a "slot number" which is very related to the "block number" but is not the same thing. The distinction between these will be covered in the following tutorial, so we defer a full discussion of how to get the "block number" until then.

## block.coinbase
In Ethereum, the "Block Coinbase" represents the address of the miner who has successfully mined a block in Proof of Work (PoW). On the other hand, Solana uses a leader-based consensus mechanism which is a combination of both Proof of History (PoH) and Proof of Stake (PoS), removing the concept of mining. Instead, a [block or slot leader](https://docs.solana.com/cluster/leader-rotation) is appointed to validate transactions and propose blocks during certain intervals, under a system known as the [leader schedule](https://docs.solana.com/cluster/leader-rotation#leader-schedule-rotation). This schedule determines who will be the block producer at a certain time.

However, presently, there's no specific way to access the address of the block leader in Solana programs.

## blockhash
We include this section for completeness, but this will soon be deprecated.

This section can be skipped with no consequence for the uninterested reader.

Solana has a [RecentBlockhashes sysvar](https://docs.rs/solana-program/1.17.2/solana_program/sysvar/recent_blockhashes/struct.RecentBlockhashes.html) that holds active recent block hashes as well as their associated fee calculators. However, this `sysvar` has been [deprecated](https://docs.rs/solana-program/1.17.3/src/solana_program/sysvar/recent_blockhashes.rs.html) and will not be supported in future Solana releases. The `RecentBlockhashes sysvar` does not provide a get method like the Clock `sysvar` does. However, `sysvar`s lacking this method can be accessed using `sysvar_name::from_account_info`.

We will also introduce some new syntax which will be explained at a later date. For now, please treat it as boilerplate:
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    /// CHECK: readonly
    pub recent_blockhashes: AccountInfo<'info>,
}
```

**Here's how we get the latest block hash in Solana:**
```rust
use anchor_lang::{prelude::*, solana_program::sysvar::recent_blockhashes::RecentBlockhashes};

// replace program id
declare_id!("H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy");

#[program]
pub mod sysvar {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        // RECENT BLOCK HASHES
        let arr = [ctx.accounts.recent_blockhashes.clone()];
        let accounts_iter = &mut arr.iter();
        let sh_sysvar_info = next_account_info(accounts_iter)?;
        let recent_blockhashes = RecentBlockhashes::from_account_info(sh_sysvar_info)?;
        let data = recent_blockhashes.last().unwrap();

        msg!("The recent block hash is: {:?}", data.blockhash);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    /// CHECK: readonly
    pub recent_blockhashes: AccountInfo<'info>,
}
```

**The test file:**
```javascript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Sysvar } from "../target/types/sysvar";

describe("sysvar", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Sysvar as Program<Sysvar>;

  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods
      .initialize()
      .accounts({
        recentBlockhashes: anchor.web3.SYSVAR_RECENT_BLOCKHASHES_PUBKEY,
      })
      .rpc();

    console.log("Transaction hash:", tx);
  });
});
```

We run the test and get the following log:

```bash
Log Messages:
  Program H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy invoke [1]
  Program log: Instruction: Initialize
  Program log: The recent block hash is: erVQHJdJ11oL4igkQwDnv7oPZoxt88j7u8DCwHkVFnC
  Program H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy consumed 46181 of 200000 compute units
  Program H52ppiSyiZyYVn1Yr9DgeUKeChktUiPwDfuuo932Uqxy success
```

<!-- ![test output](https://static.wixstatic.com/media/935a00_f883ec55d8ea49e399403e176d6265d7~mv2.png/v1/fill/w_1480,h_200,al_c,lg_1,q_85,enc_auto/935a00_f883ec55d8ea49e399403e176d6265d7~mv2.png) -->

We can see the latest block hash. Note that because we are deploying to our local node, the block hash we get is that of our local node and not the Solana mainnet.


In terms of time structuring, Solana operates on a fixed timeline divided into slots, with each slot being the portion of time allocated to a leader for proposing a block. These slots are further organized into epochs, which are pre-defined periods during which the leader schedule remains unchanged.

## block.gaslimit
Solana has a per-block compute unit [limit of 48 million](https://github.com/solana-labs/solana/issues/29492). Each transaction is by default limited to 200,000 compute units, though it can be raised to 1.4 million compute units (we will discuss this in a later tutorial, though you can [see an example here](https://solanacookbook.com/references/basic-transactions.html#how-to-change-compute-budget-fee-priority-for-a-transaction)).


It is not possible to access this limit from a Rust program.

## block.basefee
In Ethereum, the basefee is dynamic per EIP-1559; it is a function of previous block utilization. In Solana, the base price of a transaction is static, so there is no need for a variable like this.

## block.difficulty
Block difficulty is a concept associated with Proof of Work (PoW) blockchains. Solana, on the other hand, operates on a Proof of History (PoH) combined with Proof of Stake (PoS) consensus mechanism, which doesn't involve the concept of block difficulty.

## block.chainid
Solana does not have a chain id because it is not an EVM compatible blockchain. The block.chainid is how Solidity smart contracts know if they are on a testnet, L2, mainnet or some other EVM compatible chain.


Solana runs separate clusters for [Devnet, Testnet, and Mainnet](https://docs.solana.com/clusters), but programs do not have a mechanism to know which one they are on. However, you can programatically adjust your code at deploy time using the Rust cfg feature to have different features depending on which cluster it is deployed to. Here is an example of [changing the program id depending on the cluster](https://solana.stackexchange.com/questions/848/how-to-have-a-different-program-id-depending-on-the-cluster).

## Learn more
This tutorial is part of our free [Solana course](https://hackmd.io/4eVoPWjpRLCqf03vK7CyVg?view).

*Originally Published February, 18, 2024*