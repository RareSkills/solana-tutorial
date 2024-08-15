# Cost of storage, maximum storage size, and account resizing in Solana

![Solana account rent](https://static.wixstatic.com/media/935a00_0d3663d61dff48bf81ac520eee9261b7~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0d3663d61dff48bf81ac520eee9261b7~mv2.jpg)
When allocating storage space, the payer must pay a certain number of SOL per byte allocated.

Solana calls this the "rent". This name is a bit misleading because it implies a monthly top-up is required, but this is not always the case. Once the rent is paid, no more payments are needed, even after the two years pass. When two years of rent are paid, the account is considered "rent exempt."

The name comes from Solana originally charging accounts in terms of bytes per year. If you only paid enough rent for half a year, your account would get erased after six months. If you paid for two years of rent in advance, the account would be "rent exempt." The account would never have to pay rent again. Nowadays, all accounts are required to be rent-exempt; you cannot pay less than 2 years of rent.

Although rent is computed on a "per byte" basis, accounts with zero data are not free; Solana still has to index them and store metadata about them.

**When accounts are initialized, the amount of rent needed is computed in the background; you don't need to calculate the rent explicitly**.

However, you do want to be able to anticipate how much storage will cost so you can design your application properly.

If you want a quick estimate, running `solana rent <number of bytes>` in the command line will give you a quick answer:
![solana rent 32](https://static.wixstatic.com/media/935a00_f4d109b3ccca403aaa4474180b6044be~mv2.png/v1/fill/w_700,h_134,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f4d109b3ccca403aaa4474180b6044be~mv2.png)
    
As mentioned earlier, allocating zero bytes is not free:
![solana rent 0](https://static.wixstatic.com/media/935a00_38db1f0b0eb9473bae5c0d6148f63c1d~mv2.png/v1/fill/w_700,h_134,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_38db1f0b0eb9473bae5c0d6148f63c1d~mv2.png)

Let's see how this fee is calculated.

The [Anchor Rent Module](https://docs.rs/solana-program/latest/solana_program/rent/index.html) gives us some constants related to rent:

+ `ACCOUNT_STORAGE_OVERHEAD`: this constant has a value of 128 (bytes) and as the name suggests, an empty account has 128 bytes of overhead.
+ `DEFAULT_EXEMPTION_THRESHOLD`: this constant has a value of 2.0 (float 64) and refers to the fact that paying two years of rent in advance makes the account exempt from paying further rent.
+ `DEFAULT_LAMPORTS_PER_BYTE_YEAR`: this constant has a value of 3,480 meaning each byte requires 3,480 lamports per year. Since we are required to pay two years worth, each byte will cost us 6,960 lamports.

The following rust program prints out how much an empty account will cost us. Note that the result matches the screenshot of the `solana rent 0` above:
```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::rent as rent_module;

declare_id!("BfMny1VwizQh89rZtikEVSXbNCVYRmi6ah8kzvze5j1S");

#[program]
pub mod rent {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let cost_of_empty_acc = rent_module::ACCOUNT_STORAGE_OVERHEAD as f64 * 
                                rent_module::DEFAULT_LAMPORTS_PER_BYTE_YEAR as f64 *
                                rent_module::DEFAULT_EXEMPTION_THRESHOLD; 

        msg!("cost to create an empty account: {}", cost_of_empty_acc);
        // 890880

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```
If we want to compute how much a non-empty account will cost, then we simply add the number of bytes to the cost of an empty account as follows:
```rust
use anchor_lang::prelude::*;
use anchor_lang::solana_program::rent as rent_module;

declare_id!("BfMny1VwizQh89rZtikEVSXbNCVYRmi6ah8kzvze5j1S");

#[program]
pub mod rent {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let cost_of_empty_acc = rent_module::ACCOUNT_STORAGE_OVERHEAD as f64 * 
                                rent_module::DEFAULT_LAMPORTS_PER_BYTE_YEAR as f64 *
                                rent_module::DEFAULT_EXEMPTION_THRESHOLD;

        msg!("cost to create an empty account: {}", cost_of_empty_acc);
        // 890,880 lamports
        
        let cost_for_32_bytes = cost_of_empty_acc + 
                                32 as f64 * 
                                rent_module::DEFAULT_LAMPORTS_PER_BYTE_YEAR as f64 *
                                rent_module::DEFAULT_EXEMPTION_THRESHOLD;

        msg!("cost to create a 32 byte account: {}", cost_for_32_bytes);
        // 1,113,600 lamports
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```
Again, note that the output of this program matches the output on the command line.

## Comparing Storage costs to ETH
At the current time of writing, ETH has a value of about $2,425. Initializing a new account costs 22,100 gas, so we can calculate the gas cost of 32 bytes to be $0.80 assuming gas costs are 15 gwei.

Currently, Solana has a price of $90 / SOL, so paying 1,113,600 lamports to initialize a 32 byte storage will cost $0.10.

However, ETH has a market capitalization 7.5x as much as SOL, so if SOL had the same market capitalization as ETH, the current price of SOL would be $675, and the 32 byte storage would cost $0.75.

Solana has a permanent inflation model that will eventually converge to 1.5% per year, so this should help reflect the fact that storage gets cheaper over time per Moore's Law, which states that transistor density for the same cost doubles every 18 months.

Remember, the translation from bytes to crypto are constants set in the protocol that a hard fork could change at any time.

## Accounts with balances below the 2 year rent except threshold are reduced until the account is deleted
A rather humorous Reddit thread of a user with a wallet account slowly getting "drained" can be read here: [https://www.reddit.com/r/solana/comments/qwin1h/my_sol_balance_in_the_wallet_is_decreasing/](https://www.reddit.com/r/solana/comments/qwin1h/my_sol_balance_in_the_wallet_is_decreasing/)

The reason for this is the wallet was below the rent exception threshold, and the Solana runtime is slowly reducing the account balance to pay for the rent.

If a wallet ends up getting deleted due to having the balance below the rent exempt threshold, it can be "resurrected" by sending more SOL to it, but if data is stored in the account, that data will be lost.

## Size limitations
When we initialize an account, we cannot initialize more than 10,240 bytes in size.

**Exercise**: create a basic storage initialization program and set `space=10241`. This is 1 byte higher than the limit. You should see an error like the following:
![solana account cannot be initialized due to exceeding size limitation](https://static.wixstatic.com/media/935a00_1292e88b8b014ff58682fbd586115fc7~mv2.png/v1/fill/w_1391,h_297,al_c,lg_1,q_90,enc_auto/935a00_1292e88b8b014ff58682fbd586115fc7~mv2.png)

## Changing the size of an account
If you need to increase the size of the account, we can use the `realloc` macro. This may be handy if the account is storing a vector and needs more space. An example is in the `increase_account_size` function and `IncreaseAccountSize` context struct which increases the size by 1,000 bytes (see the ALL CAPS comment in the code below):

```rust
use anchor_lang::prelude::*;
use std::mem::size_of;

declare_id!("GLKUcCtHx6nkuDLTz5TNFrR4tt4wDNuk24Aid2GrDLC6");

#[program]
pub mod basic_storage {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }

    pub fn increase_account_size(ctx: Context<IncreaseAccountSize>) -> Result<()> {
        Ok(())
    }
}


#[derive(Accounts)]
pub struct IncreaseAccountSize<'info> {

    #[account(mut,
              // ***** 1,000 BYTE INCREMENT IS OVER HERE *****
              realloc = size_of::<MyStorage>() + 8 + 1000,
              realloc::payer = signer,
              realloc::zero = false,
              seeds = [],
              bump)]
    pub my_storage: Account<'info, MyStorage>,
    
    #[account(mut)]
    pub signer: Signer<'info>,

    pub system_program: Program<'info, System>,
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
When increasing the size of the account, be sure to set `realloc::zero = false` (in the code above) if you do not want the account data erased. If you want the account data to be set to all zeros, use `realloc::zero = true`. You do not need to change the test. The macro will handle this behind the scenes for you.

**Exercise**: Initialize an account in the test, then call the `increase_account_size` function. View the account size with `solana account <addr>`. In the command line. You will need to do this with the local validator so the account persists.

## Maximum Solana account size
The maximum account size increase per realloc is 10240. The maximum size an account can be in Solana is 10 MB.

## Anticipating the cost of deploying a program
The bulk of the cost of deploying a Solana program comes from paying rent for storing the bytecode. The bytecode is stored in a separate account from the address returned from `anchor deploy`.

The screenshot below shows how to obtain this information:
![cost of deploying a program](https://static.wixstatic.com/media/935a00_b2995dfebb704e33a45a91b0e2d980de~mv2.png/v1/fill/w_1480,h_568,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_b2995dfebb704e33a45a91b0e2d980de~mv2.png)

A simple hello world program currently costs a little over 2.47 SOL to deploy. The cost can be significantly reduced by writing raw Rust code instead of using the Anchor framework, but we don't recommend doing that until you fully understand all the security risks Anchor eliminates by default.

## Learn more with RareSkills
See our [Solana developer course](https://www.rareskills.io/solana-tutorial) to learn more.

*Originally Published February, 28, 2024*