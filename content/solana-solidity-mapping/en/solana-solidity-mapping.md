# Creating "mappings" and "nested mapping" in Solana

!["Mappings" and "Nested Mappings" in Solana](https://static.wixstatic.com/media/935a00_fcc8fb7861f344a6b54b33647dc34ef2~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_fcc8fb7861f344a6b54b33647dc34ef2~mv2.jpg)

In the previous tutorials, the `seeds=[]` parameter was always empty. If we put data into it, it behaves like a key or keys in a Solidity mapping.

Consider the following example:
```solidity
contract ExampleMapping {

    struct SomeNum {
        uint64 num;
    }

    mapping(uint64 => SomeNum) public exampleMap;

    function setExampleMap(uint64 key, uint64 val) public {
        exampleMap[key] = SomeNum(val);
    }
}
```
We now create a Solana Anchor program `example_map`.

## Initializing a mapping: Rust
At first, we will only show the initialization step because it will introduce some new syntax we need to explain.
```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("DntexDPByFxpVeBSjd6nLqQQSqZmSaDkP8TUbcJ9jAgt");

#[program]
pub mod example_map {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, key: u64) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(key: u64)]
pub struct Initialize<'info> {

    #[account(init,
              payer = signer,
              space = size_of::<Val>() + 8,
              seeds=[&key.to_le_bytes().as_ref()],
              bump)]
    val: Account<'info, Val>,
    
    #[account(mut)]
    signer: Signer<'info>,
    
    system_program: Program<'info, System>,
}

#[account]
pub struct Val {
    value: u64,
}
```
Here's how you can think of the map:

The seeds parameter `key` in `&key.to_le_bytes().as_ref()` can be thought of as a "key" to the map similar to the Solidity construction:
```solidity
mapping(uint256 => uint256) myMap;
myMap[key] = val
```
The unfamiliar parts of the code are `#[instruction(key: u64)]` and `seeds=[&key.to_le_bytes().as_ref()]`.

### seeds = [&key.to_le_bytes().as_ref()]
The items in `seeds` are expected to be bytes. However, we are passing in a `u64` which is not of type bytes. To convert it to bytes, we use `to_le_bytes()`. The "le" means "[little endian](https://www.freecodecamp.org/news/what-is-endianness-big-endian-vs-little-endian/)". Seeds do not have to be encoded as little endian bytes, we just chose that for this example. Big endian works too as long as you are consistent. To convert to big endian, we would have used `to_be_bytes()`.

### #[instruction(key: u64)]
In order to "pass" the function argument `key` in `initialize(ctx: Context<Initialize>, key: u64)` we need to use the `instruction` macro, otherwise our `init` macro has no way to "see" the `key` argument from `initialize`.

## Initializing a mapping: Typescript
The code below shows how to initialize the account:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ExampleMap } from "../target/types/example_map";

describe("example_map", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ExampleMap as Program<ExampleMap>;

  it("Initialize mapping storage", async () => {
    const key = new anchor.BN(42);

    const [seeds, _bump] = [key.toArrayLike(Buffer, "le", 8)];
    let valueAccount = anchor.web3.PublicKey.findProgramAddressSync(
      seeds,
      program.programId,
    );

    await program.methods.initialize(key).accounts({val: valueAccount}).rpc();
  });
});
```
The code `key.toArrayLike(Buffer, "le", 8)` specifies that we are trying to create a bytes buffer of size 8 bytes using the value from `key`. We chose `8` bytes because our key is 64 bits, and 64 bits is 8 bytes. The "le" is little endian so that we match the Rust code.

Each "value" in the mapping is a separate account and must be initialized separately.

## Set a mapping: Rust
The additional Rust code we need to set the value. All the syntax here should be familiar.

```rust
// inside the #[program] module
pub fn set(ctx: Context<Set>, key: u64, val: u64) -> Result<()> {
    ctx.accounts.val.value = val;
    Ok(())
}

//...

#[derive(Accounts)]
#[instruction(key: u64)]
pub struct Set<'info> {
    #[account(mut)]
    val: Account<'info, Val>,
}
```

## Set and read a mapping: Typescript
Because we derive the account address where the value is stored in the client (Typescript), we read and write from it just like we do with accounts that have the `seeds` array empty. The syntax for [reading the Solana account data](https://www.rareskills.io/post/solana-read-account-data) and writing is identical to previous tutorials:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ExampleMap } from "../target/types/example_map";

describe("example_map", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ExampleMap as Program<ExampleMap>;

  it("Initialize and set value", async () => {
    const key = new anchor.BN(42);
    const value = new anchor.BN(1337);

    const seeds = [key.toArrayLike(Buffer, "le", 8)];
    let valueAccount = anchor.web3.PublicKey.findProgramAddressSync(
      seeds,
      program.programId,
    )[0];

    await program.methods.initialize(key).accounts({val: valueAccount}).rpc();

    // set the account
    await program.methods.set(key, value).accounts({val: valueAccount}).rpc();

    // read the account back
    let result = await program.account.val.fetch(valueAccount);

    console.log(`the value ${result.value} was stored in ${valueAccount.toBase58()}`);
  });
});
```

## Clarifying "nested mappings"
In languages like Python or JavaScript, a true nested mapping is a hashmap that points to another hash map.


In Solidity however, "nested mappings" are only a single map with multiple keys behaving as if they are one key.

In a "true" nested mapping, you can provide only the first key and get another hashmap returned to you.

Solidity "nested mappings" are not "true" nested mappings: you cannot supply one key and get a map back: you must provide all the keys and get the final result.

If you use seeds to simulate nested mappings similar to Solidity, you will face the same restriction. You must supply all of the seeds — Solana will not accept only one seed.

## Initializing a nested mapping: Rust
The `seeds` array can hold as many items as we like, similar to a nested mapping in Solidity. It is of course subject to compute limits imposed on each of transaction. The code to do the initialization and setting are shown below.

We do not need any special syntax to do this, it's just a matter of taking more function arguments and putting more items in `seeds`, so we will show the complete code without further explanation.

### Rust nested mapping
```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("DntexDPByFxpVeBSjd6nLqQQSqZmSaDkP8TUbcJ9jAgt");

#[program]
pub mod example_map {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, key1: u64, key2: u64) -> Result<()> {
        Ok(())
    }

    pub fn set(ctx: Context<Set>, key1: u64, key2: u64, val: u64) -> Result<()> {
        ctx.accounts.val.value = val;
        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(key1: u64, key2: u64)] // new key args added
pub struct Initialize<'info> {

    #[account(init,
              payer = signer,
              space = size_of::<Val>() + 8,
              seeds=[&key1.to_le_bytes().as_ref(), &key2.to_le_bytes().as_ref()], // 2 seeds
              bump)]
    val: Account<'info, Val>,
    
    #[account(mut)]
    signer: Signer<'info>,
    
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
#[instruction(key1: u64, key2: u64)] // new key args added
pub struct Set<'info> {
    #[account(mut)]
    val: Account<'info, Val>,
}

#[account]
pub struct Val {
    value: u64,
}
```

### Typescript nested mapping
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ExampleMap } from "../target/types/example_map";

describe("example_map", () => {
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ExampleMap as Program<ExampleMap>;

  it("Initialize and set value", async () => {
    // we now have two keys
    const key1 = new anchor.BN(42);
    const key2 = new anchor.BN(43);
    const value = new anchor.BN(1337);

    // seeds has two values
    const seeds = [key1.toArrayLike(Buffer, "le", 8), key2.toArrayLike(Buffer, "le", 8)];
    let valueAccount = anchor.web3.PublicKey.findProgramAddressSync(
      seeds,
      program.programId,
    )[0];

    // functions now take two keys
    await program.methods.initialize(key1, key2).accounts({val: valueAccount}).rpc();
    await program.methods.set(key1, key2, value).accounts({val: valueAccount}).rpc();

    // read the account back
    let result = await program.account.val.fetch(valueAccount);
    console.log(`the value ${result.value} was stored in ${valueAccount.toBase58()}`);
  });
});
```
**Exercise**: Modify the above code to form a nested mapping that takes three keys.

## Initializing more than one map
A straightforward way to accomplish having more than one map is to add another variable to the `seeds` array and treat it as a way to "index" the first map, second map, and so forth.

The following code shows an example of initializing `which_map` which only holds one key.
```rust
#[derive(Accounts)]
#[instruction(which_map: u64, key: u64)]
pub struct InitializeMap<'info> {

    #[account(init,
              payer = signer,
              space = size_of::<Val1>() + 8,
              seeds=[&which_map.to_le_bytes().as_ref(), &key.to_le_bytes().as_ref()],
              bump)]
    val: Account<'info, Val1>,

    #[account(mut)]
    signer: Signer<'info>,

    system_program: Program<'info, System>,
}
```
**Exercise**: Complete the Rust and Typescript code to create a program that has two mappings: the first one with a single key and second one with two keys. Think about how to turn a two level map into a single level map when the first map is specified.

## Learn Solana with RareSkills
See our [Solana course](http://rareskills.io/solana-tutorial) to see the rest of our Solana tutorials.

*Originally Published February, 27, 2024*