# Native Solana: Essential Security Checks

In our previous native Solana tutorials, we skipped security checks to keep examples short and focused on the core topics.

In this tutorial, we'll cover essential security checks for native Solana programs: validating account ownership, verifying sysvar and program IDs, requiring signers, enforcing writable accounts, reloading state after CPIs, and handling token account dust attacks.

## **Validate account ownership**

Before using an account's data, check that its owner matches the expected program ID. Otherwise, an attacker can pass an account they control with malicious data.

In Anchor, defining an account with `Account<'info, T>` automatically checks that the account is owned by your program. For external accounts, you can add the `#[account(owner = <ID>)]` attribute to enforce ownership by a specific program ID.

For example, say we have a `Config` account that controls withdrawals. If we don't verify the config is owned by our program, an attacker can pass a fake account with fake data and withdraw when they shouldn't.

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{account_info::AccountInfo, program_error::ProgramError};

#[derive(BorshDeserialize, BorshSerialize)]
struct Config { withdraw_cap: u64 }

pub fn withdraw(config: &AccountInfo, amount: u64) -> Result<(), ProgramError> {
    // Missing check: config.owner == program_id
    let cfg = Config::try_from_slice(&config.try_borrow_data()?)?;
    
    if amount <= cfg.withdraw_cap {
        // proceed to transfer funds...
    }
    
    Ok(())
}
```

To fix this, we check that the config account is owned by our program before deserializing or using its data.

```rust
use solana_program::{account_info::AccountInfo, program_error::ProgramError, pubkey::Pubkey};

pub fn withdraw(config: &AccountInfo, amount: u64, program_id: &Pubkey) -> Result<(), ProgramError> {
		// Check that the config account is owned by our program
    if config.owner != program_id { return Err(ProgramError::IncorrectProgramId); }

    // Rest of the function...

    Ok(())
}
```

## **Validate Sysvar and Program IDs**

When you need a sysvar like the Clock sysvar or a system program, always verify that it's the real one. An attacker can pass fake accounts with manipulated data.

In Anchor programs, this is enforced by using the `Sysvar<'info, Clock>` or `Program<'info, System>` types, which handle the checks for you. But we have to manually check the IDs in native programs.

In this example, the `clock` account passed to `withdraw_timelock` is expected to be the Clock sysvar, but without verification, an attacker can pass a fake clock account with a manipulated timestamp to withdraw early.

```rust
// Vulnerable: fake sysvar allows time manipulation
use solana_program::{account_info::AccountInfo, program_error::ProgramError, clock::Clock};

#[derive(BorshDeserialize, BorshSerialize)]
struct TimeLock {
    unlock_time: i64,
    amount: u64,
}

pub fn withdraw_timelock(timelock: &AccountInfo, clock: &AccountInfo) -> Result<(), ProgramError> {
    // Missing: verify clock.key == sysvar::clock::ID
    let timelock = TimeLock::try_from_slice(&timelock.try_borrow_data()?)?;
    
    // Attacker can pass fake clock account with manipulated timestamp
    let clock = Clock::from_account_info(clock)?;
    
    if clock.unix_timestamp >= timelock.unlock_time {
        // Process early withdrawal with fake timestamp
    }
    Ok(())
}
```

As a fix, always ensure that that sysvar and program account addresses matches

```rust
use solana_program::{sysvar, program_error::ProgramError, account_info::AccountInfo};
// Rest of the code...

pub fn withdraw_timelock(timelock: &AccountInfo, clock: &AccountInfo) -> Result<(), ProgramError> {
    // Fix: verify clock.key == sysvar::clock::ID
    if clock.key != &sysvar::clock::ID { 
        return Err(ProgramError::InvalidAccountData); 
    }

    // Rest of the function...
    
    Ok(())
}
```

### **Real life example**

The $320M Wormhole bridge exploit on Solana (Feb 2022) happened because the program did not ensure that the provided system program address matched the actual System Program address. The attacker passed in a fake system account that bypassed signature checks, allowing them to mint 120,000 wETH (wrapped ETH) without authorization.

This is why Solana programs must always verify that system accounts and sysvars match their official IDs. [You can read CertiK's full analysis here.](https://www.certik.com/resources/blog/wormhole-bridge-exploit-incident-analysis)

## **Require Signers**

When your program restricts an action to a specific authority (e.g., an admin), it's not enough to check that the account's public key matches the expected one. An attacker can include the real admin account in the transaction as a non-signer and pass that check. You must also verify that the account actually signed the transaction (`admin.is_signer`), which proves the private key owner approved it. In Anchor, the `#[account(signer)]` attribute handles this for you.

Here's a vulnerable native Rust example where a function updates the withdraw cap in a config account:

```rust
// Vulnerable: checks admin key but not that admin actually signed

#[derive(BorshDeserialize, BorshSerialize)]
struct Config { admin: Pubkey, withdraw_cap: u64 }

pub fn update_withdraw_cap(config: &AccountInfo, admin: &AccountInfo, new_cap: u64) -> Result<(), ProgramError> {
    let mut cfg = Config::try_from_slice(&config.try_borrow_data()?)?;
    
    // Missing check: require admin.is_signer
    
    if *admin.key != cfg.admin { return Err(ProgramError::InvalidAccountData); }
    cfg.withdraw_cap = new_cap;
    cfg.serialize(&mut &mut config.try_borrow_mut_data()?[..])?;
    Ok(())
}
```

To fix this, require the admin to be a signer and match the stored admin key:

```rust
pub fn update_withdraw_cap(config: &AccountInfo, admin: &AccountInfo, new_cap: u64) -> Result<(), ProgramError> {
    let mut cfg = Config::try_from_slice(&config.try_borrow_data()?)?;

    // Ensure admin is a signer
    if !admin.is_signer { return Err(ProgramError::MissingRequiredSignature); }

    // Ensure admin is the expected admin
    if *admin.key != cfg.admin { return Err(ProgramError::InvalidAccountData); }
    
    cfg.withdraw_cap = new_cap;
    cfg.serialize(&mut &mut config.try_borrow_mut_data()?[..])?;
    Ok(())
}
```

## Require Writable Accounts for Data or Lamport Modifications

If your program needs to modify an account's lamports or data, the client must mark that account as writable in the transaction, and your program should verify it is writable. If the account isn't marked writable, attempting to modify it will cause the transaction to fail with an error (e.g., "Readonly account changed").

To mark the account as writable in a TypeScript client, we set its `isWritable` flag to `true` during the transaction construction.

```tsx
// web3.js: set isWritable = true for accounts you will modify
import { TransactionInstruction, PublicKey } from "@solana/web3.js";

const ix = new TransactionInstruction({
  programId,
  keys: [
    { pubkey: userAccount, isSigner: false, isWritable: true }, // needs mutation
    { pubkey: payer, isSigner: true, isWritable: false },
  ],
  data: Buffer.from([]),
});
```

Our program can check if an account is writable using the `is_writable` field of the `AccountInfo` struct.

```rust
pub fn update_user_balance(user_account: &AccountInfo) -> Result<(), ProgramError> {
		// Check if supplied user account is writable before proceeding
    if !user_account.is_writable {
        return Err(ProgramError::InvalidAccountData);
    }
    
    // Rest of the function...
    Ok(())
}
```

In Anchor, you enforce this with `#[account(mut)]`, which checks `is_writable = true` before your function runs.

## **Reload Account State After Each CPI**

A CPI can modify account data. Your program must read the account again after a CPI call before making decisions based on it.

Here's what can happen without reloading (using a native Rust program code):

```rust
// Vulnerable: using stale account data after CPI
use solana_program::{
    account_info::AccountInfo, 
    program::{invoke},
    instruction::Instruction,
    program_error::ProgramError
};
use borsh::BorshDeserialize;

#[derive(BorshDeserialize)]
struct VaultState {
    balance: u64,
    is_locked: bool,
}

pub fn withdraw_after_cpi_vulnerable(
    vault: &AccountInfo,
    external_program: &AccountInfo,
    cpi_instruction: Instruction,
    amount: u64
) -> Result<(), ProgramError> {
    // Read vault state before CPI
    let vault_state = VaultState::try_from_slice(&vault.try_borrow_data()?)?;
    
    // Perform CPI that might modify the vault
    invoke(&cpi_instruction, &[vault.clone(), external_program.clone()])?;
    
    // Vulnerable: using stale vault_state after CPI
    // The CPI might have changed vault.is_locked to true
    if !vault_state.is_locked && vault_state.balance >= amount {
        // Process withdrawal with stale data
    }
    
    Ok(())
}
```

In this code, the CPI might modify the vault's state (e.g., set `is_locked` to `true`), but we're still using the old `vault_state` that was read before the CPI. This creates a time-of-check to time-of-use (TOCTOU) vulnerability, meaning that the state we checked is no longer the state we're acting upon.

To fix this, always reload account data after a CPI:

```rust
use solana_program::{
    account_info::AccountInfo,
    program::invoke,
    instruction::Instruction,
    program_error::ProgramError
};

pub fn withdraw_after_cpi_safe(
    vault: &AccountInfo,
    external_program: &AccountInfo,
    cpi_instruction: Instruction,
    amount: u64
) -> Result<(), ProgramError> {
    // Read vault state before CPI
    let vault_state_before = VaultState::try_from_slice(&vault.try_borrow_data()?)?;
    
    // Perform CPI
    invoke(&cpi_instruction, &[vault.clone(), external_program.clone()])?;
    
    // Reload vault state after CPI
    let vault_state_after = VaultState::try_from_slice(&vault.try_borrow_data()?)?;
    
    // Use the fresh data for decisions
    if !vault_state_after.is_locked && vault_state_after.balance >= amount {
        // Process withdrawal with current data
    }
    
    Ok(())
}
```

## Burn Remaining Tokens Before Closing Token Accounts

Imagine you're building a liquid staking protocol. Users deposit SOL and receive receipt tokens proving their stake. When they unstake, they burn their receipt tokens to get their SOL back. Your program stores each user's receipt tokens in a PDA token account (owned by your program), and after burning the receipt tokens, you close the PDA token account to return the rent back to the user.

The problem arises when the program assumes the PDA token account only contains the user's legitimate receipt tokens. An attacker can exploit this by sending just 1 receipt token of the same mint (a "dust attack") directly to the victim's PDA token account before they unstake.

When the legitimate user tries to unstake, the program burns only the amount of receipt tokens the user originally received when they staked. But since the attacker's 1 dust receipt token is still in the account, the balance is not zero. The close account operation then fails because the SPL Token Program requires an account's balance to be exactly zero before closing. Since our program doesn't check for or burn any remaining tokens before closing, the attacker's dust token stays there. This will fail every time, permanently locking the user's staked SOL and causing a DoS.

Here's an example:

```rust
// Vulnerable: assumes token account balance is always zero
use solana_program::{
    account_info::AccountInfo,
    entrypoint::ProgramResult,
    program::invoke_signed,
    pubkey::Pubkey,
};
use spl_token::instruction::{burn, close_account};

pub fn unstake_vulnerable(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    amount: u64, // assume this to be the amount of receipt tokens the user originally received when they staked
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let receipt_token_account = next_account_info(account_info_iter)?;
    let receipt_mint = next_account_info(account_info_iter)?;
    let vault_authority = next_account_info(account_info_iter)?;
    let user = next_account_info(account_info_iter)?;
    let token_program = next_account_info(account_info_iter)?;
    
    // Burn the user's original receipt tokens
    invoke_signed(
        &burn(
            token_program.key,
            receipt_token_account.key,
            receipt_mint.key,
            vault_authority.key,
            &[],
            amount,
        )?,
        &[receipt_token_account.clone(), receipt_mint.clone(), vault_authority.clone()],
        &[&vault_seeds],
    )?;
    
    // Missing: check if any dust tokens remain before closing
    invoke_signed(
        &close_account(
            token_program.key,
            receipt_token_account.key,
            user.key,
            vault_authority.key,
            &[],
        )?,
        &[receipt_token_account.clone(), user.clone(), vault_authority.clone()],
        &[&vault_seeds],
    )?;
    
    Ok(())
}
```

In this code, the program burns only the `amount` the user originally staked. If an attacker sent 1 dust token beforehand, it remains in the account. The `close_account` call fails because the SPL Token Program requires the balance to be exactly zero, permanently locking the user's staked SOL.

This vulnerability applies to both native Solana programs and Anchor programs when closing SPL token accounts.

To fix this, fetch the actual on-chain balance and burn any remaining tokens:

```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint::ProgramResult,
    program::invoke_signed,
    program_pack::Pack,
    pubkey::Pubkey,
};
use spl_token::{instruction::{burn, close_account}, state::Account as TokenAccount};

pub fn unstake_safe(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let receipt_token_account = next_account_info(account_info_iter)?;
    let receipt_mint = next_account_info(account_info_iter)?;
    let vault_authority = next_account_info(account_info_iter)?;
    let user = next_account_info(account_info_iter)?;
    let token_program = next_account_info(account_info_iter)?;
    
    // Fetch actual on-chain balance
    let token_account = TokenAccount::unpack(&receipt_token_account.try_borrow_data()?)?;
    
    // Burn all tokens (user's original amount + any dust)
    if token_account.amount > 0 {
        invoke_signed(
            &burn(
                token_program.key,
                receipt_token_account.key,
                receipt_mint.key,
                vault_authority.key,
                &[],
                token_account.amount, // Burns everything including dust
            )?,
            &[receipt_token_account.clone(), receipt_mint.clone(), vault_authority.clone()],
            &[&vault_seeds],
        )?;
    }
    
    // Safe to close now
    invoke_signed(
        &close_account(
            token_program.key,
            receipt_token_account.key,
            user.key,
            vault_authority.key,
            &[],
        )?,
        &[receipt_token_account.clone(), user.clone(), vault_authority.clone()],
        &[&vault_seeds],
    )?;
    
    Ok(())
}
```

## Frontrunning token account or contract creation

The System Program's `create_account` instruction fails if the target account already has lamports. Since PDA addresses are deterministic (derived from known seeds), an attacker can calculate the address before the account is created and send 1 lamport to it. When the program later tries to create the account at that address, `create_account` fails because the account already has a balance — causing a DoS.

For example, say we have a vault program where each user gets a PDA vault derived from their public key:

```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint::ProgramResult,
    program::invoke_signed,
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::Sysvar,
};

pub fn initialize_vault(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let payer = &accounts[0];
    let vault = &accounts[1];
    let system_program = &accounts[2];

    let (vault_pda, bump) = Pubkey::find_program_address(
        &[b"vault", payer.key.as_ref()],
        program_id,
    );

    let rent = Rent::get()?;
    let space = 48; // vault data size
    let lamports = rent.minimum_balance(space);

    // Vulnerable: fails if attacker sent lamports to vault_pda beforehand
    invoke_signed(
        &system_instruction::create_account(
            payer.key,
            &vault_pda,
            lamports,
            space as u64,
            program_id,
        ),
        &[payer.clone(), vault.clone(), system_program.clone()],
        &[&[b"vault", payer.key.as_ref(), &[bump]]],
    )?;

    Ok(())
}
```

The attacker simply derives the same PDA address (`seeds = ["vault", victim_pubkey]`), sends 1 lamport to it via a normal SOL transfer, and the victim can never initialize their vault.

To fix this, check if the account already has lamports. If it does, skip `create_account` and instead use `transfer` (to top up the rent), `allocate` (to reserve the account's data space), and `assign` (to set the account's owner to your program) separately — these instructions don't fail on accounts that already have a balance:

```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint::ProgramResult,
    program::invoke_signed,
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::Sysvar,
};

pub fn initialize_vault_safe(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
) -> ProgramResult {
    let payer = &accounts[0];
    let vault = &accounts[1];
    let system_program = &accounts[2];

    let (vault_pda, bump) = Pubkey::find_program_address(
        &[b"vault", payer.key.as_ref()],
        program_id,
    );
    let signer_seeds = &[b"vault", payer.key.as_ref(), &[bump]];

    let rent = Rent::get()?;
    let space = 48;
    let required_lamports = rent.minimum_balance(space);

    if vault.lamports() > 0 {
        // Account already has lamports (possibly from an attacker).
        // Top up to rent-exempt minimum if needed.
        let deficit = required_lamports.saturating_sub(vault.lamports());
        if deficit > 0 {
            invoke_signed(
                &system_instruction::transfer(payer.key, &vault_pda, deficit),
                &[payer.clone(), vault.clone()],
                &[signer_seeds],
            )?;
        }

        // Allocate space and assign ownership to our program
        invoke_signed(
            &system_instruction::allocate(&vault_pda, space as u64),
            &[vault.clone()],
            &[signer_seeds],
        )?;
        invoke_signed(
            &system_instruction::assign(&vault_pda, program_id),
            &[vault.clone()],
            &[signer_seeds],
        )?;
    } else {
        // No lamports — safe to use create_account
        invoke_signed(
            &system_instruction::create_account(
                payer.key,
                &vault_pda,
                required_lamports,
                space as u64,
                program_id,
            ),
            &[payer.clone(), vault.clone(), system_program.clone()],
            &[signer_seeds],
        )?;
    }

    // Rest of the code...

    Ok(())
}
```

By checking `vault.lamports() > 0` first, we handle both cases: normal creation (no prior lamports) and the frontrunning scenario (attacker sent lamports). The `allocate` and `assign` instructions work on accounts that already have a balance, so the attacker's griefing attempt has no effect.

In Anchor, using [`init_if_needed`](https://rareskills.io/post/init-if-needed-anchor) instead of `init` handles this scenario.