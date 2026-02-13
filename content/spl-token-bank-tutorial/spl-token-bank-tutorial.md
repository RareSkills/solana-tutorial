# Basic Bank Tutorial with SPL Tokens and Anchor

In this tutorial, we’ll build a simple bank program on Solana with the basic features you'd expect from a regular bank. Users can create accounts, check balances, deposit funds, and withdraw their funds when needed. The deposited SOL will be stored in a bank PDA, owned by our program. 

Here’s a Solidity representation of what we’re trying to build with Anchor. 

In the Solidity code, we define a `User` and `Bank` struct to hold state. These mirror the separate accounts we would create in a Solana program. The `initialize` function in the code sets the bank’s total deposit to zero, but in a Solana program this would involve deploying and initializing the bank account itself. Similarly, the `createUserAccount` function in the Solidity code just updates storage, while in Solana it would create and initialize a new user account on-chain. Also note how deposits and withdrawals update both the user and bank state, with custom errors enforcing the rules when balances are insufficient or invalid.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract BasicBank {
    // Custom errors
    error ZeroAmount();
    error InsufficientBalance();
    error Overflow();
    error Underflow();
    error InsufficientFunds();
    error UnauthorizedAccess();

    // Bank struct to track total deposits across all users
    struct Bank {
        uint256 totalDeposits;
    }

    // User account struct to store individual balances
    struct UserAccount {
        address owner;
        uint256 balance;
    }

    // The bank state
    Bank public bank;

    // Mapping from user address to their user account
    mapping(address => UserAccount) public userAccounts;

    // Initialize the bank
    function initialize() external {
        bank.totalDeposits = 0;
    }

    // Create a user account
    function createUserAccount() external {
        // Ensure account doesn't already exist
        require(userAccounts[msg.sender].owner == address(0), "Account already created");

        // Initialize the user account
        userAccounts[msg.sender].owner = msg.sender;
        userAccounts[msg.sender].balance = 0;
    }

    // Deposit ETH into the bank
    function deposit(uint256 amount) external payable {
        // Ensure amount is greater than zero
        if (amount == 0) {
            revert ZeroAmount();
        }

        // Ensure account exists and is owned by caller
        if (userAccounts[msg.sender].owner != msg.sender) {
            revert UnauthorizedAccess();
        }
        // Ensure the correct amount was sent
        require(msg.value == amount, "Amount mismatch");

        // Update user balance with checks for overflow
        uint256 newUserBalance = userAccounts[msg.sender].balance + amount;
        if (newUserBalance < userAccounts[msg.sender].balance) {
            revert Overflow();
        }
        userAccounts[msg.sender].balance = newUserBalance;

        // Update bank total deposits with checks for overflow
        uint256 newTotalDeposits = bank.totalDeposits + amount;
        if (newTotalDeposits < bank.totalDeposits) {
            revert Overflow();
        }
        bank.totalDeposits = newTotalDeposits;
    }

    // Withdraw ETH from the bank
    function withdraw(uint256 amount) external {
        // Ensure amount is greater than zero
        if (amount == 0) {
            revert ZeroAmount();
        }

        // Ensure account exists and is owned by caller
        if (userAccounts[msg.sender].owner != msg.sender) {
            revert UnauthorizedAccess();
        }
        // Check if user has enough balance
        if (userAccounts[msg.sender].balance < amount) {
            revert InsufficientBalance();
        }

        // Update user balance with checks for underflow
        uint256 newBalance = userAccounts[msg.sender].balance - amount;
        userAccounts[msg.sender].balance = newBalance;

        // Update bank total deposits with checks for underflow
        uint256 newTotalDeposits = bank.totalDeposits - amount;
        if (newTotalDeposits > bank.totalDeposits) {
            revert Underflow();
        }
        bank.totalDeposits = newTotalDeposits;

        // Transfer ETH to the user
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }

    // Get the balance of the caller's bank account
    function getBalance() external view returns (uint256) {
        // Ensure account exists and is owned by caller
        if (userAccounts[msg.sender].owner != msg.sender) {
            revert UnauthorizedAccess();
        }
        return userAccounts[msg.sender].balance;
    }
}

```

## Creating the Basic Bank program

Before diving into the code, here's how our basic bank program will work. We'll need the following functionalities:

- A way to set up and initialize the bank
- A way for users to create their own accounts
- A way for users to deposit funds
- A way for users to withdraw funds
- A way to check balances

We'll need the following storage:

- A central account to track the total deposits across all users
- Individual accounts, that can be created by users to track their balance and ownership

All of these elements come together to form our basic bank program.

Now, let's dive in.

### Bank PDA and user account

Create a new Anchor project called `basic_bank`, and add the program code below. The program defines two instructions: `initialize`, which creates a bank PDA to hold deposits, and `create_user_account`, which sets up a user-specific PDA to track each user's total deposits. The actual SOL deposited by users will be stored in the bank PDA.

We have a dedicated account for users because in Solana, all program data must exist as its own account. Unlike Ethereum where we could use a mapping to store deposits (`mapping(address => deposited_amount)`), [Solana requires explicit account allocation for storing state](https://rareskills.io/post/solana-initialize-account). Hence, we create a user_account PDA to store both the user's address and their deposit amount.

```rust

use anchor_lang::prelude::*;
use anchor_lang::solana_program::rent::Rent;
use anchor_lang::solana_program::system_instruction;
use anchor_lang::solana_program::program as solana_program;

declare_id!("u9tNA22L1oRZyF3RKoPVUYTAc1zCYSC5BySKFddZnfN"); // RUN ANCHOR SYNC TO UPDATE YOUR PROGRAM ID

#[program]
pub mod basic_bank {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        // Initialize the bank account
        let bank = &mut ctx.accounts.bank;
        bank.total_deposits = 0;

        msg!("Bank initialized");
        Ok(())
    }

    pub fn create_user_account(ctx: Context<CreateUserAccount>) -> Result<()> {
        // Initialize the user account
        let user_account = &mut ctx.accounts.user_account;
        user_account.owner = ctx.accounts.user.key();
        user_account.balance = 0;

        msg!("User account created for: {:?}", user_account.owner);
        Ok(())
    }
}
```

Let's define the account structs for our basic bank program. This structs are used by both the `initialize` and `create_user_account` functions, they are:

- `Initialize` ****struct, which contains:
    - `bank`: The new bank account being created to track total deposits
    - `payer`: The account paying for transaction fees and rent
    - `system_program`: Required to create the new account
- `CreateUserAccount` struct contains:
    - `bank`: Reference to the main bank account
    - `user_account`: A PDA derived from the user's public key that stores their balance
    - `user`: The signer who owns this account and pays for its creation
    - `system_program`: Required for account creation
- Finally, we define a `Bank` and a `UserAccount` struct that define the data structure of their respective accounts. The `Bank` struct contains a `total_deposits` field to track all deposits in the bank account, while the `UserAccount` struct stores a user’s public key and their individual balance in the user account PDA.

```rust

// ACCOUNT STRUCT TO CREATE THE BANK PDA TO STORE TO
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + Bank::INIT_SPACE)] // discriminator + u64
    pub bank: Account<'info, Bank>,

    #[account(mut)]
    pub payer: Signer<'info>,

    pub system_program: Program<'info, System>,
}

// ACCOUNT STRUCT FOR CREATING INDIVIDUAL USER ACCOUNT
#[derive(Accounts)]
pub struct CreateUserAccount<'info> {
    #[account(mut)]
    pub bank: Account<'info, Bank>,

    #[account(
        init,
        payer = user,
        space = 8 + UserAccount::INIT_SPACE, // discriminator + pubkey + u64
        seeds = [b"user-account", user.key().as_ref()],
        bump
    )]
    pub user_account: Account<'info, UserAccount>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub system_program: Program<'info, System>,
}

// BANK ACCOUNT TO TRACK TOTAL DEPOSITS ACROSS ALL USERS 
#[account]
#[derive(InitSpace)]
pub struct Bank { 
    pub total_deposits: u64,
}

// USER-SPECIFIC ACCOUNT TO TRACK INDIVIDUAL USER BALANCES
#[account]
#[derive(InitSpace)]
pub struct UserAccount {
    pub owner: Pubkey,
    pub balance: u64,
}

```

Next, add the test below to initialize and create a user account. 

The test does the following:

1. Generates a keypair for the bank account
2. Uses the provider's wallet (our default Anchor wallet) as the signer for all operations
3. Sets the test amount values (1 SOL for deposit, 0.5 SOL for withdrawal)
4. Derives a PDA for the user account for the signer's public key
5. Calls `initialize` to set up the bank account and verifies it starts with a zero balance.
6. Calls `createUserAccount` to initialize a user-specific PDA and asserts that the user’s address and balance are correctly recorded.

```tsx
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey } from "@solana/web3.js";
import { assert } from "chai";
import { BasicBank } from "../target/types/basic_bank";

describe("basic_bank", () => {
  // Configure the client to use the local cluster
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.BasicBank as Program<BasicBank>;
  const provider = anchor.AnchorProvider.env();

  // Generate a new keypair for the bank account
  const bankAccount = Keypair.generate();

  // Use provider's wallet as the signer
  const signer = provider.wallet;

  // Test deposit amount
  const depositAmount = new anchor.BN(1_000_000_000); // 1 SOL in lamports
  const withdrawAmount = new anchor.BN(500_000_000); // 0.5 SOL in lamports

  // Find PDA for user accounts
  const [userAccountPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from("user-account"), signer.publicKey.toBuffer()],
    program.programId,
  );

  it("Initializes the bank account", async () => {
    // Initialize the bank account
    const tx = await program.methods
      .initialize()
      .accounts({
        bank: bankAccount.publicKey,
        payer: signer.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([bankAccount])
      .rpc();

    console.log("Initialize transaction signature", tx);

    // Fetch the bank account data
    const bankData = await program.account.bank.fetch(bankAccount.publicKey);

    // Verify the bank is initialized correctly
    assert.equal(bankData.totalDeposits.toString(), "0");
  });

  it("Creates a user account", async () => {
    // Create user account for the signer
    const tx = await program.methods
      .createUserAccount()
      .accounts({
        bank: bankAccount.publicKey,
        userAccount: userAccountPDA,
        user: signer.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    console.log("Create user account transaction signature", tx);

    // Fetch the user account data
    const userAccountData = await program.account.userAccount.fetch(userAccountPDA);

    // Verify the user account is set correctly
    assert.equal(userAccountData.owner.toString(), signer.publicKey.toString());
    assert.equal(userAccountData.balance.toString(), "0");
  });
});

```

Run the test, it should pass. 

![A test showing that the account creation was successful](https://r2media.rareskills.io/SolanaSPLTokenBank/image7.png)

We will now implement code to deposit funds into the bank, and write a test for it. 

### Deposit SOL to bank

Add the deposit function below to our program. 

It does the following:

1. Validates that the deposit amount is greater than zero
2. Creates and executes a System  `transfer` instruction (via CPI) to pull SOL from the user's wallet to the bank account
3. Safely increment the user’s account balance by adding the deposited amount to their `user_account` PDA
4. Updates the bank's total deposits with the same amount
5. Logs the deposited amount and user address

```rust

    pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
        // Ensure deposit amount is greater than zero
        require!(amount > 0, BankError::ZeroAmount);

        let user = &ctx.accounts.user.key();
        let bank = &ctx.accounts.bank.key();

        // Transfer SOL from user to bank account using System Program
        let transfer_ix = system_instruction::transfer(user, bank, amount);
        solana_program::invoke(
            &transfer_ix,
            &[
                ctx.accounts.user.to_account_info(),
                ctx.accounts.bank.to_account_info(),
            ],
        )?;

        // Update user balance
        let user_account = &mut ctx.accounts.user_account;
        user_account.balance = user_account
            .balance
            .checked_add(amount)
            .ok_or(BankError::Overflow)?;

        // Update bank total deposits
        let bank = &mut ctx.accounts.bank;
        bank.total_deposits = bank
            .total_deposits
            .checked_add(amount)
            .ok_or(BankError::Overflow)?;

        msg!("Deposited {} lamports for {}", amount, user);
        Ok(())
    }

```

Take note of how we use the system `transfer` instruction to pull SOL from the user wallet into the bank, we'll revisit this pattern later. 

Now, add the `Deposit` account struct

```rust
#[derive(Accounts)]
pub struct Deposit<'info> {
    #[account(mut)]
    pub bank: Account<'info, Bank>,

    #[account(
        mut,
        seeds = [b"user-account", user.key().as_ref()],
        bump,
        constraint = user_account.owner == user.key() @ BankError::UnauthorizedAccess // Ensure the signer owns the account
    )]
    pub user_account: Account<'info, UserAccount>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub system_program: Program<'info, System>,
}
```

Add the custom `BankError`

```rust

#[error_code]
pub enum BankError {
    #[msg("Amount must be greater than zero")]
    ZeroAmount,

    #[msg("Insufficient balance for withdrawal")]
    InsufficientBalance,

    #[msg("Arithmetic overflow")]
    Overflow,

    #[msg("Arithmetic underflow")]
    Underflow,

    #[msg("Insufficient funds in the bank account")]
    InsufficientFunds,

    #[msg("Unauthorized access to user account")]
    UnauthorizedAccess,
}

```

Now update the program test with the test block below.

The deposit test does the following:

1. It records initial SOL balances of both user and bank accounts
2. Submits a deposit transaction of 1 SOL from the user to the bank
3. Verifies both user account's balance record and bank's total deposit record are updated correctly with the new deposit amount
4. Check that the bank balance increased and user balance decreased appropriately (accounting for transaction fees) to confirm actual SOL transfers occurred
5. Finally, it logs all balance changes

```tsx

  it("Deposits funds into the bank", async () => {
    // Get initial SOL balances
    const initialUserBalance = await provider.connection.getBalance(signer.publicKey);
    const initialBankBalance = await provider.connection.getBalance(bankAccount.publicKey);

    console.log(`Initial user SOL balance: ${initialUserBalance / 1e9} SOL`);
    console.log(`Initial bank SOL balance: ${initialBankBalance / 1e9} SOL`);

    // Deposit funds into the bank
    const tx = await program.methods
      .deposit(depositAmount)
      .accounts({
        bank: bankAccount.publicKey,
        userAccount: userAccountPDA,
        user: signer.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    console.log("Deposit transaction signature", tx);

    // Get the user's account balance
    const userAccountData = await program.account.userAccount.fetch(userAccountPDA);

    // Verify the tracked balance is correct
    assert.equal(userAccountData.balance.toString(), depositAmount.toString());

    // Verify bank total tracked deposits
    const bankData = await program.account.bank.fetch(bankAccount.publicKey);
    assert.equal(bankData.totalDeposits.toString(), depositAmount.toString());

    // Get final SOL balances
    const finalUserBalance = await provider.connection.getBalance(signer.publicKey);
    const finalBankBalance = await provider.connection.getBalance(bankAccount.publicKey);

    console.log(`Final user SOL balance: ${finalUserBalance / 1e9} SOL`);
    console.log(`Final bank SOL balance: ${finalBankBalance / 1e9} SOL`);

    // Check actual SOL transfers (accounting for tx fees)
    assert.isTrue(finalBankBalance > initialBankBalance);

    // User balance should be reduced by deposit amount + some tx fees
    assert.isTrue(finalUserBalance < initialUserBalance - Number(depositAmount));
    assert.isTrue(finalUserBalance > initialUserBalance - Number(depositAmount) - 10000); // Account for a reasonable tx fees
});
```

Run the test, and it passes. 

![A screenshot showing a deposit was successful](https://r2media.rareskills.io/SolanaSPLTokenBank/image1.png)

Next, we will add a function to retrieve a user's deposited amount in our bank.

### Get user balance

The `get_balance` function below retrieves and returns the lamport balance for a user. 

```rust

    pub fn get_balance(ctx: Context<GetBalance>) -> Result<u64> {
        // Get user account
        let user_account = &ctx.accounts.user_account;
        let balance = user_account.balance;

        msg!("Balance for {}: {} lamports", user_account.owner, balance);
        Ok(balance)
    }
    
```

Add the `GetBalance` account struct. 

```rust

#[derive(Accounts)]
pub struct GetBalance<'info> {
    pub bank: Account<'info, Bank>,

    #[account(
        seeds = [b"user-account", user.key().as_ref()],
        bump,
        constraint = user_account.owner == user.key() @ BankError::UnauthorizedAccess // Ensure the signer owns the account
    )]
    pub user_account: Account<'info, UserAccount>,

    pub user: Signer<'info>,
}

```

Add the test for the function. 

The test invokes our Anchor program’s **getBalance** function and asserts that the returned balance is equal to the amount we deposited earlier (1 SOL). 

```tsx

  it("Retrieves user balance", async () => {
    // Get the user's balance
    const balance = await program.methods
      .getBalance()
      .accounts({
        bank: bankAccount.publicKey,
        userAccount: userAccountPDA,
        user: signer.publicKey,
      })
      .view(); // The .view() method in Anchor is used to call instructions that only read data (view functions) without submitting an actual transaction

    // Verify the balance is correct
    assert.equal(balance.toString(), depositAmount.toString());

    console.log(`User balance: ${Number(balance) / 1e9} SOL`);
  });
```

Run the test, it passes. 

![A test showing the the balance was read successfully](https://r2media.rareskills.io/SolanaSPLTokenBank/image2.png)

Now, we’ll add a withdraw implementation and write test for it. 

### Withdraw balance from bank: Success case

Add the code below to withdraw user deposits from our bank. 

The withdraw function does the following:

1. Validates that withdrawal amount is greater than zero
2. Checks if user has sufficient balance for the withdrawal
3. Updates user account balance and bank's total deposits using checked arithmetic
4. Calculates minimum balance needed to keep the bank account rent-exempt
5. Determines safe transfer amount from the bank that preserves the rent-exempt minimum
6. Transfers SOL from the bank to the user's wallet using direct lamport manipulation (since our program owns the bank account)
7. Logs the withdrawal details

```rust

    pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
        // Ensure withdraw amount is greater than zero
        require!(amount > 0, BankError::ZeroAmount);

        // Get accounts
        let bank = &mut ctx.accounts.bank;
        let user_account = &mut ctx.accounts.user_account;
        let user = ctx.accounts.user.key();

        // Check if the user has enough balance
        require!(
            user_account.balance >= amount,
            BankError::InsufficientBalance
        );

        // Update user balance
        user_account.balance = user_account
            .balance
            .checked_sub(amount)
            .ok_or(BankError::Underflow)?;

        // Update bank total deposits
        bank.total_deposits = bank
            .total_deposits
            .checked_sub(amount)
            .ok_or(BankError::Underflow)?;

        // The bank account must keep enough lamports to stay rent-exempt,
        // otherwise the runtime will garbage-collect it.
        let rent = Rent::get()?;
        let bank_account_info = ctx.accounts.bank.to_account_info();
        let minimum_balance = rent.minimum_balance(bank_account_info.data_len());

        // Only transfer what the bank can afford after reserving rent.
        // Cap at the requested amount so we never send more than asked.
        let available_lamports = bank_account_info.lamports();
        let transfer_amount = amount.min(available_lamports.saturating_sub(minimum_balance));

        // Transfer SOL: subtract from bank account and add to user wallet
        **bank_account_info.try_borrow_mut_lamports()? -= transfer_amount;
        **ctx.accounts.user.try_borrow_mut_lamports()? += transfer_amount;

        msg!("Withdrawn {} lamports for {}", amount, user);
        Ok(())
    }

```

Recall from the **deposit** function, we use the System Program’s transfer function to pull SOL from the user’s account into the bank. We do this because the System Program owns all regular wallets (like EOAs in Ethereum) and has the permission to modify their balances.

![A screenshot showing the transfer CPI instructions](https://r2media.rareskills.io/SolanaSPLTokenBank/image3.png)

But in the **withdraw** function, we can modify the lamport balances of both the bank and the user account PDA directly. [This is because our program owns both PDAs](https://www.rareskills.io/post/solana-account-owner) (we deployed them).

![A screenshot showing that the balance of the lamports are modified directly](https://r2media.rareskills.io/SolanaSPLTokenBank/image4.png)

Now add the `Withdraw` account struct

```rust

#[derive(Accounts)]
pub struct Withdraw<'info> {
    #[account(mut)]
    pub bank: Account<'info, Bank>,

    #[account(
        mut,
        seeds = [b"user-account", user.key().as_ref()],
        bump,
        constraint = user_account.owner == user.key() @ BankError::UnauthorizedAccess
    )]
    pub user_account: Account<'info, UserAccount>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub system_program: Program<'info, System>,
}

```

Add the test for the withdraw function. 

It does the following:

1. It records initial balances of the user account and gets SOL balances of both the user and the bank
2. Submits a withdrawal transaction of 0.5 SOL from the bank to the user
3. Verifies both user account balance and bank's total deposits are updated correctly after withdrawal
4. Check that the user balance increased (accounting for transaction fees) to confirm actual SOL transfers occurred
5. Finally, we log all balance changes for the transaction

```tsx

  it("Withdraws funds from the bank", async () => {
    // Get the initial balance
    const userAccountData = await program.account.userAccount.fetch(userAccountPDA);
    const initialBalance = userAccountData.balance;

    // Get initial SOL balances
    const initialUserBalance = await provider.connection.getBalance(signer.publicKey);
    const initialBankBalance = await provider.connection.getBalance(bankAccount.publicKey);

    console.log(`Initial user SOL balance: ${initialUserBalance / 1e9} SOL`);
    console.log(`Initial bank SOL balance: ${initialBankBalance / 1e9} SOL`);

    // Withdraw funds from the bank
    const tx = await program.methods
      .withdraw(withdrawAmount)
      .accounts({
        bank: bankAccount.publicKey,
        userAccount: userAccountPDA,
        user: signer.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .rpc();

    console.log("Withdraw transaction signature", tx);

    // Get the new balance
    const updatedUserAccountData = await program.account.userAccount.fetch(userAccountPDA);
    const newBalance = updatedUserAccountData.balance;

    // Verify the balance is correct
    const expectedBalance = initialBalance.sub(withdrawAmount);
    assert.equal(newBalance.toString(), expectedBalance.toString());

    // Verify bank total deposits
    const bankData = await program.account.bank.fetch(bankAccount.publicKey);
    assert.equal(bankData.totalDeposits.toString(), expectedBalance.toString());

    // Get final SOL balances
    const finalUserBalance = await provider.connection.getBalance(signer.publicKey);
    const finalBankBalance = await provider.connection.getBalance(bankAccount.publicKey);

    console.log(`Final user SOL balance: ${finalUserBalance / 1e9} SOL`);
    console.log(`Final bank SOL balance: ${finalBankBalance / 1e9} SOL`);

    // Check actual SOL transfers
    // User balance should increase by withdraw amount (minus tx fees)
    // Since the user pays tx fees, the final balance might be slightly less than expected
    assert.isTrue(finalUserBalance < initialUserBalance + Number(withdrawAmount));
    assert.isTrue(finalUserBalance > initialUserBalance - 10000); // Allow for reasonable tx fees

    // Bank balance should decrease by withdraw amount
    assert.isTrue(finalBankBalance <= initialBankBalance);
  });

```

Now run the test. It passes. 

![A screenshot showing the funds were withdrawn successfully](https://r2media.rareskills.io/SolanaSPLTokenBank/image5.png)

### Withdraw balance from bank: Failure case

Let’s add one more test block to confirm user’s cannot withdraw more than what they deposited. 

Add the following code, it does the following:

1. It attempts to withdraw an excessive amount (10 SOL) from the bank
2. Expects the transaction to fail with an insufficient balance error
3. Uses a try/catch block to capture the expected failure and logs the actual error message received
4. Verifies the error contains either "insufficient balance" text
5. Fails the test if no error is thrown (which would indicate a missing validation — what we don’t want)

```tsx

  it("Prevents users from withdrawing more than their balance", async () => {
    // Try to withdraw more than the balance
    const excessiveWithdrawAmount = new anchor.BN(10_000_000); // 10 SOL

    try {
      await program.methods
        .withdraw(excessiveWithdrawAmount)
        .accounts({
          bank: bankAccount.publicKey,
          userAccount: userAccountPDA,
          user: signer.publicKey,
          systemProgram: anchor.web3.SystemProgram.programId,
        })
        .rpc();

      // If we reach here, the test failed
      assert.fail("Should have thrown an error for insufficient balance");
    } catch (error) {
      // Log the actual error
      console.log("Error received:", error.toString());

      // Check for multiple possible error messages that could indicate insufficient balance
      const errorMsg = error.toString().toLowerCase();
      assert.isTrue(
        errorMsg.includes("insufficient balance") ||
        errorMsg.includes("0x7d3")
      );
    }
  });
  
```

Finally, run the test. It passes. 

![A screenshot showing users cannot withdraw more than their balance](https://r2media.rareskills.io/SolanaSPLTokenBank/image6.png)

This concludes our basic bank program. 

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*
