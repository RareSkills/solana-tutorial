# Understanding Account Ownership in Solana: Transferring SOL out of a PDA

![Hero image showing Solona account ownership](https://static.wixstatic.com/media/935a00_46458c8f748f4697835ec9db35ab9657~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_46458c8f748f4697835ec9db35ab9657~mv2.jpg)

The owner of an account in Solana is able to reduce the SOL balance, write data to the account, and change the owner.

Here is the summary of account ownership in Solana:

1. The `system program` owns wallets and keypair accounts that haven't been assigned ownership to a program (initialized).
2. The BPFLoader owns programs.
3. A program owns [Solana PDAs](https://www.rareskills.io/post/solana-pda). It can also own keypair accounts if ownership has been transferred to the program (this is what happens during initialization).

We now examine the implications of these facts.

## The system program owns keypair accounts
To illustrate this, let's look at our Solana wallet address using the Solana CLI and inspect its metadata:

![Solona metadata : owner](https://static.wixstatic.com/media/935a00_18047d3cc76b443f8ed1db16968ed951~mv2.png/v1/fill/w_812,h_224,al_c,lg_1,q_85,enc_auto/935a00_18047d3cc76b443f8ed1db16968ed951~mv2.png)

Observe that the owner is not our address, but rather an account with address `111...111`. This is the system program, the same system program that moves SOL around as we saw in earlier tutorials.

**Only the owner of an account has the ability to modify the data in it**

This includes reducing the lamport data (you do not need to be the owner to increase the lamport data of another account as we will see later).

Although you "own" your wallet in some metaphysical sense, you do not directly have the ability to write data into it or reduce the lamport balance because, from Solana runtime perspective, you are not the owner.

The reason you are able to spend SOL in your wallet is because you possess the private key that generated the address, or public key. When the `system program` recognizes that you have produced a valid signature for the public key, then it will recognize your request to spend the lamports in the account as legitimate, then spend them according to your instructions.

However, the system program does not offer a mechanism for a signer to directly write data to the account.

The account showed in the example above is a keypair account, or what we might consider a "regular Solana wallet." The system program is the owner of keypair accounts.

## PDAs and keypair accounts initialized by programs are owned by the program
The reason programs can write to PDAs or keypair accounts that were created outside the program but initialized by the program, is because the program owns them.

We will explore initialization more closely when we discuss the re-initialization attack, but for now, the important takeaway is that **initializing an account changes the owner of the account from the system program the program.**

To illustrate this, consider the following program that initializes a PDA and a keypair account. The Typescript test will console log the owner before and after the initialization transaction.

If we try to ascertain the owner of an address that does not exist, we get a `null`.

Here is the Rust code:

```rust
use anchor_lang::prelude::*;

declare_id!("C2ZKJPhNiCM6CqTneGUXJoE4o6YhMzNUes3q5WNcH3un");

#[program]
pub mod owner {
    use super::*;

    pub fn initialize_keypair(ctx: Context<InitializeKeypair>) -> Result<()> {
        Ok(())
    }

    pub fn initialize_pda(ctx: Context<InitializePda>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeKeypair<'info> {
    #[account(init, payer = signer, space = 8)]
    keypair: Account<'info, Keypair>,
    #[account(mut)]
    signer: Signer<'info>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct InitializePda<'info> {
    #[account(init, payer = signer, space = 8, seeds = [], bump)]
    pda: Account<'info, Pda>,
    #[account(mut)]
    signer: Signer<'info>,
    system_program: Program<'info, System>,
}

#[account]
pub struct Keypair();

#[account]
pub struct Pda();
```

Here is the Typescript code:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Owner } from "../target/types/owner";

async function airdropSol(publicKey, amount) {
  let airdropTx = await anchor.getProvider().connection.requestAirdrop(publicKey, amount * anchor.web3.LAMPORTS_PER_SOL);
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

describe("owner", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Owner as Program<Owner>;

  it("Is initialized!", async () => {
    console.log("program address", program.programId.toBase58());    
    const seeds = []
    const [pda, bump_] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("owner of pda before initialize:",
    await anchor.getProvider().connection.getAccountInfo(pda));

    await program.methods.initializePda().accounts({pda: pda}).rpc();

    console.log("owner of pda after initialize:",
    (await anchor.getProvider().connection.getAccountInfo(pda)).owner.toBase58());

    let keypair = anchor.web3.Keypair.generate();

    console.log("owner of keypair before airdrop:",
    await anchor.getProvider().connection.getAccountInfo(keypair.publicKey));

    await airdropSol(keypair.publicKey, 1); // 1 SOL
   
    console.log("owner of keypair after airdrop:",
    (await anchor.getProvider().connection.getAccountInfo(keypair.publicKey)).owner.toBase58());
    
    await program.methods.initializeKeypair()
      .accounts({keypair: keypair.publicKey})
      .signers([keypair]) // the signer must be the keypair
      .rpc();

    console.log("owner of keypair after initialize:",
    (await anchor.getProvider().connection.getAccountInfo(keypair.publicKey)).owner.toBase58());
 
  });
});
```

The tests work as follows:
1. It predicts the address of the PDA and queries the owner. It gets `null`.
2. It calls `initializePDA` then queries the owner. It gets the address of the program.
3. It generates a keypair account and queries the owner. It gets `null`.
4. It airdrops SOL to the keypair account. Now the owner is the system program, just like a normal wallet.
5. It calls `initializeKeypair` then queries the owner. It gets the address of the program.

The test result screenshot is below:

![test result : Is initialized](https://static.wixstatic.com/media/935a00_55cd8f204aa64cf2a105fc745f5a18fa~mv2.png/v1/fill/w_1259,h_267,al_c,lg_1,q_85,enc_auto/935a00_55cd8f204aa64cf2a105fc745f5a18fa~mv2.png)

This is how the program is able to write data to accounts: it owns them. During initialization, the program takes ownership over the account.

**Exercise**: Modify the test to print out the address of the keypair and the pda. Then use the Solana CLI to inspect who the owner is for those accounts. It should match what the test prints. Make sure the `solana-test-validator` is running in the backgorund so you can use the CLI.

## The BPFLoaderUpgradeable owns programs
Let's use the Solana CLI to determine the owner of our program:

![Solona metadata : Owner: the BPFLoaderUpgradable](https://static.wixstatic.com/media/935a00_396ec64ed6bf429fb84fd1252b62cdb6~mv2.png/v1/fill/w_1397,h_349,al_c,lg_1,q_90,enc_auto/935a00_396ec64ed6bf429fb84fd1252b62cdb6~mv2.png)

The wallet that deployed the program is not the owner of it. The reason Solana programs are able to be upgraded by the deploying wallet is because the BpfLoaderUpgradeable is able to write new bytecode to the program, and it will only accept new bytecode from a predesignated address: the address that originally deployed the program.

When we deploy (or upgrade) a program, we are actually making a call to the BPFLoaderUpgradeable program, as can be seen in the logs:

```bash
  Signature: 2zBBEPWsMvf8t7wkNEDqfHJKw83aBMgwGi3G9uZ6m9qG9t4kjJA2wFEP84dkKCjiCdbh54xeEDYFeDcNS7FkyLEw  
  Status: Ok  
  Log Messages:
    Program 11111111111111111111111111111111 invoke [1]
    Program 11111111111111111111111111111111 success
    Program BPFLoaderUpgradeab1e11111111111111111111111 invoke [1]
    Program 11111111111111111111111111111111 invoke [2]
    Program 11111111111111111111111111111111 success
    Deployed program C2ZKJPhNiCM6CqTneGUXJoE4o6YhMzNUes3q5WNcH3un
    Program BPFLoaderUpgradeab1e11111111111111111111111 success
Transaction executed in slot 34:
```

<!-- ![logs : the BPFLoaderUpgradable](https://static.wixstatic.com/media/935a00_b29171cb29a34ae49c47c92571e49af9~mv2.png/v1/fill/w_1395,h_281,al_c,lg_1,q_90,enc_auto/935a00_b29171cb29a34ae49c47c92571e49af9~mv2.png) -->

## Programs can transfer ownership of owned accounts
This is a feature you will probably not use very often, but here is the code to do it.

Rust:

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;
use anchor_lang::system_program;

declare_id!("Hxj38tktrD7YcSvKRxVrYQfxptkZd7NVbmrRKvLxznyA");


#[program]
pub mod change_owner {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn change_owner(ctx: Context<ChangeOwner>) -> Result<()> {
        let account_info = &mut ctx.accounts.my_storage.to_account_info();
        
	// assign is the function to transfer ownership
	account_info.assign(&system_program::ID);

	// we must erase all the data in the account or the transfer will fail
        let res = account_info.realloc(0, false);

        if !res.is_ok() {
            return err!(Err::ReallocFailed);
        }

        Ok(())
    }
}

#[error_code]
pub enum Err {
    #[msg("realloc failed")]
    ReallocFailed,
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

#[derive(Accounts)]
pub struct ChangeOwner<'info> {
    #[account(mut)]
    pub my_storage: Account<'info, MyStorage>,
}

#[account]
pub struct MyStorage {
    x: u64,
}
```

Typescript:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ChangeOwner } from "../target/types/change_owner";

import privateKey from '/Users/jeffreyscholz/.config/solana/id.json';

describe("change_owner", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.ChangeOwner as Program<ChangeOwner>;

  it("Is initialized!", async () => {
    const deployer = anchor.web3.Keypair.fromSecretKey(Uint8Array.from(privateKey));

    const seeds = []
    const [myStorage, _bump] = anchor.web3.PublicKey.findProgramAddressSync(seeds, program.programId);

    console.log("the storage account address is", myStorage.toBase58());

    await program.methods.initialize().accounts({myStorage: myStorage}).rpc();
    await program.methods.changeOwner().accounts({myStorage: myStorage}).rpc();
    
    // after the ownership has been transferred
    // the account can still be initialized again
    await program.methods.initialize().accounts({myStorage: myStorage}).rpc();
  });
});
```

Here are some things we want to call attention to:
- After transferring the account, the data must be erased in the same transaction. Otherwise, we could insert data into owned accounts of other programs. This is the `account_info.realloc(0, false)`; code. The `false` means don't zero out the data, but it makes no difference because there is no data anymore.
- Transferring account ownership does not permanently remove the account, it can be initialized again as the tests show.

Now that we clearly understand that programs own PDAs and keypair accounts initialized by them, the interesting and useful thing we can do is transfer SOL out of them.

## Transferring SOL out of a PDA: Crowdfund example
Below we show the code for a barebones crowdfunding app. The function of interest is the `withdraw` function where the program transfer lamports out of the PDA and to the withrdawer.

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program;
use std::mem::size_of;
use std::str::FromStr;

declare_id!("BkthFL8LV2V2MxVgQtA9tT5goeeJhUdxRPahzavqHPFZ");

#[program]
pub mod crowdfund {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let initialized_pda = &mut ctx.accounts.pda;
        Ok(())
    }

    pub fn donate(ctx: Context<Donate>, amount: u64) -> Result<()> {
        let cpi_context = CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            system_program::Transfer {
                from: ctx.accounts.signer.to_account_info().clone(),
                to: ctx.accounts.pda.to_account_info().clone(),
            },
        );

        system_program::transfer(cpi_context, amount)?;

        Ok(())
    }

    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
        ctx.accounts.pda.sub_lamports(amount)?;
        ctx.accounts.signer.add_lamports(amount)?;

        // in anchor 0.28 or lower, use the following syntax:
        // **ctx.accounts.pda.to_account_info().try_borrow_mut_lamports()? -= amount;
        // **ctx.accounts.signer.to_account_info().try_borrow_mut_lamports()? += amount;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,

    #[account(init, payer = signer, space=size_of::<Pda>() + 8, seeds=[], bump)]
    pub pda: Account<'info, Pda>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Donate<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,

    #[account(mut)]
    pub pda: Account<'info, Pda>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(mut, address = Pubkey::from_str("5jmigjgt77kAfKsHri3MHpMMFPo6UuiAMF19VdDfrrTj").unwrap())]
    pub signer: Signer<'info>,

    #[account(mut)]
    pub pda: Account<'info, Pda>,
}

#[account]
pub struct Pda {}
```
Because the program owns the PDA, it can directly deduct the lamport balance from the account.

When we transfer SOL as part of a normal wallet transaction, we don't deduct the lamport balance directly as we are not the owner of the account. The system program owns the wallet, and will deduct the lamport balance if it sees a valid signature on a transaction requesting it to do so.

In this case, the program owns the PDA, and therefore can directly deduct lamports from it.

Some other items in the code worth calling attention to:
- We hardcoded who can withdraw from the PDA using the constraint `#[account(mut, address = Pubkey::from_str("5jmigjgt77kAfKsHri3MHpMMFPo6UuiAMF19VdDfrrTj").unwrap())]`. This checks that the address for that account matches the one in the string. For this code to work, we also needed to import `use std::str::FromStr;`. To test this code, change the address in the string to yours from `solana address`.
- With Anchor 0.29, we can use the syntax `ctx.accounts.pda.sub_lamports(amount)?;` and `ctx.accounts.signer.add_lamports(amount)?;`. For earlier versions of Anchor, use **`ctx.accounts.pda.to_account_info().try_borrow_mut_lamports()? -= amount;`** **and** `ctx.accounts.signer.to_account_info().try_borrow_mut_lamports()? += amount;`.
- You don't need to own the account you are transferring lamports to.

Here is the accompanying Typescript code:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Crowdfund } from "../target/types/crowdfund";

describe("crowdfund", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.Crowdfund as Program<Crowdfund>;

  it("Is initialized!", async () => {
    const programId = await program.account.pda.programId;

    let seeds = [];
    let pdaAccount = anchor.web3.PublicKey.findProgramAddressSync(seeds, programId)[0];

    const tx = await program.methods.initialize().accounts({
      pda: pdaAccount
    }).rpc();

    // transfer 2 SOL
    const tx2 = await program.methods.donate(new anchor.BN(2_000_000_000)).accounts({
      pda: pdaAccount
    }).rpc();

    console.log("lamport balance of pdaAccount",
    await anchor.getProvider().connection.getBalance(pdaAccount));

    // transfer back 1 SOL
    // the signer is the permitted address
    await program.methods.withdraw(new anchor.BN(1_000_000_000)).accounts({
      pda: pdaAccount
    }).rpc();

    console.log("lamport balance of pdaAccount",
    await anchor.getProvider().connection.getBalance(pdaAccount));

  });
});
```

**Exercise**: try to add more lamports to the receiving address than you withdraw from the PDA. i.e. change the code to the following:

```rust
ctx.accounts.pda.sub_lamports(amount)?;
// sneak in an extra lamport
ctx.accounts.signer.add_lamports(amount + 1)?;
```
The runtime should block you.

Note that withdrawing the lamport balance below the rent-exempt threshold will result in the account getting closed. If there is data in the account, that will be erased. As such, programs should track how much SOL is required for rent exemption before withdrawing SOL unless they don't care about the account getting erased.

## Learn more with RareSkills
See our [Solana tutorial](https://www.rareskills.io/solana-tutorial) for the complete list of topics.

*Originally Published March, 7, 2024*