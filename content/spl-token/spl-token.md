# How the SPL Token Works

Solana Program Library Token (SPL Token) is Solana's standard for tokens: how to create tokens and how they should behave. It is Solana’s equivalent to Ethereum’s token standards like ERC-20 (fungible tokens), and ERC-721 (NFTs). 

Unlike Ethereum, which uses separate smart contracts for each token standard, all SPL tokens in Solana use the same program. This means that all tokens on Solana share the same underlying logic, with token-specific parameters set during creation rather than by a different program code. The SPL Token program contains only the logic, while all token data is stored separately. This is consistent with how Solana separates logic from state into separate accounts.

Here's one way to think about SPL tokens in contrast to Ethereum: on Ethereum, you typically deploy a new smart contract (like ERC-20) for each unique token. On Solana, rather than deploying new code, you interact with this single SPL program, which contains all the instructions needed for defining tokens, minting, transfers, approvals, and burning.

On Ethereum, each token is its own smart contract with custom code, which means USDC might handle approvals differently than DAI; this has advantages in terms of flexibility, but can also lead to unexpected behavior. On Solana, every SPL token uses the same transfer function, the same approval system, and the same security checks.

This article explains SPL token concepts and how Solana separates token logic from token data. It covers:

- How Solana’s token architecture differs from Ethereum,
- The three key accounts that make SPL tokens work,
- Why Solana uses one program for all tokens, and
- How Solana tracks user token balances.

In the next article, *Creating SPL Tokens with Anchor*, we will show how to create and transfer SPL tokens.

To understand how all this works in practice, we start by looking at the SPL Token program itself. We’ll sometimes refer to it simply as the token program, but both mean the same thing.

## **Token Program**

The SPL Token program is the core on-chain program responsible for managing SPL token functionality. It contains the logic to create SPL tokens and defines how they behave. The SPL Token program lives at a fixed address: **`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`.**

The Token Program owns all accounts that store the state of SPL tokens (we will introduce these accounts as we progress). This ownership means the Token Program is the only program that can modify the data in these accounts.

For readers who need a refresher on Solana's account ownership model, see [Understanding Account Ownership in Solana](https://rareskills.io/post/solana-account-owner).

Next, we will discuss the different accounts associated with SPL tokens, which are the Mint Account, and the Token Account and Associated Token Account (ATA). Each account plays a specific role in accounting for and transferring tokens.

## **Mint account**

Every individual SPL token has one unique Mint account which stores the global information about that token. It holds data such as the token’s total supply, number of decimals and which addresses (if any) have the authority to mint tokens and freeze accounts (i.e. blacklisting). As mentioned above, the token's core logic remains in the SPL Token Program.

Each mint account is unique and is created via the SPL Token Program when initializing a new SPL token. When we refer to a token address in Solana, we also mean its mint account address, because they’re the same. For example, here are the token addresses (mint accounts) for USDC and USDT respectively: [EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v](https://explorer.solana.com/address/EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v) and [Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB](https://explorer.solana.com/address/Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB).

![Solana block explorer showing USD coin](https://r2media.rareskills.io/SPLToken/image4.png)

![Solana block explorer showing USDT](https://r2media.rareskills.io/SPLToken/image2.png)

The diagram below shows the relationship between the Token program and mint accounts. 

![a diagram showing the accounts the Token Program owns](https://r2media.rareskills.io/SPLToken/image8.png)

### Mint account details

Like all Solana accounts, the mint account has standard metadata fields, which are:

- Lamports ([for rent](https://rareskills.io/post/solana-account-rent))
- Owner (which is the Token Program address in this case)
- Executable status (a value of `false` as it's for data storage)

For more on Solana account see [Initializing Accounts in Solana and Anchor](https://rareskills.io/post/solana-initialize-account).  

In addition to these standard fields, the [mint account contains specific data fields](https://github.com/solana-program/token/blob/main/interface/src/state.rs#L16) that define the token itself:

- `mint_authority`: Address allowed to create new tokens.
- `freeze_authority`: Address allowed to freeze token accounts holding this token.
- `decimals`: Number of decimal places (0-9) the token uses.
- `supply`: Total number of tokens created.
- `is_initialized`: Boolean flag preventing reinitialization.

While the mint account stores token information, it doesn't track individual user balances, that's handled by separate accounts we'll discuss later.

The diagram below shows the mint account properties. 

![a table showing the fields inside the mint account](https://r2media.rareskills.io/SPLToken/image5.png)

The mint and freeze authorities of a mint account (as shown above) are assigned during its creation and are usually set to the transaction signer’s address. We will see the instruction for this in the “Token program instruction” section. 

One important aspect of mint accounts is how they control the supply of a token. This is handled through the mint authority field. We explain this below. 

### How to Set a Maximum Token Supply

SPL tokens implement maximum supply through permission removal rather than explicit limits. This design choice stems from fundamental differences in how Solana and Ethereum handle state. 

A mint account data doesn't have an explicit "max supply" field. Hence, to create a fixed supply, you disable the mint authority by setting it to None (nil value). This permanently **disables the mint authority**, as it assigns it to no one. Since no account holds the authority to mint more tokens, the current total supply becomes the fixed maximum.

In Ethereum, the traditional way to limit the total supply of a token is to explicitly store the number somewhere and block minting that exceeds this number. SPL tokens do not have a notion of “total supply limit” and thus do not store such a number anywhere.

With SPL, if we want a total supply of 1 million tokens, we mint all 1 million to holders and then disable the mint authority. Alternatively, as we will see in a later tutorial, we could also make a separate program the mint authority and have that program stop minting tokens after the supply cap is reached.

## Token Accounts and Associated Token Accounts (ATAs)

As mentioned before, the mint account only stores information about the token itself. To track individual user balances, Solana uses separate accounts called Token Accounts.

### Token Accounts

Token Accounts are Solana accounts that store a user's token balance, the mint address the account is associated with, and the owner who can authorize transfers, along with other fields that we will cover in detail later.

By design, a user can have multiple Token Accounts for the same token, created at different addresses. This creates a challenge as stated by the Solana Program Library documentation:

*"A user may own arbitrarily many token accounts belonging to the same mint which makes it difficult for other users to know which account they should send tokens to."*

This means a user’s balance for one token can be split across several accounts instead of sitting in a single place.

For example:

- Alice has 5 tokens in one token account and 15 in another token account. Both token accounts belong to the same mint, so together she owns 20 tokens of that mint.
- However, if Bob wants to send Alice more tokens, he has no easy way of knowing which account she prefers to receive them in.

This is what the SPL documentation means by the challenge of “arbitrarily many token accounts.” The balances are not redundant copies, but rather **pieces of the total balance spread across multiple accounts**.

To address this problem, Solana introduced Associated Token Accounts (ATAs).

### Associated Token Accounts (ATAs)

Unlike regular Token Accounts where users can have multiple accounts per mint, ATAs are special Token Accounts (with a deterministic rule for finding the address) that enforce a one-to-one relationship between a user wallet address and a mint. This ensures that:

1. Each user has exactly one predictable ATA per token type
2. Any application can easily find a user's token balance without prior knowledge because the address is deterministic. We explain how below.

This article focuses primarily on ATAs as they have become the standard approach for managing tokens in Solana due to the challenges with regular Token Addresses discussed above. 

### How are ATA addresses derived?

An ATA address is a [Program Derived Address (PDA)](https://www.rareskills.io/post/solana-pda), deterministically derived from two inputs:

1. The address of the user/signer’s wallet (the intended authority).
2. The address of the token's mint account.

We can compare Associated Token Account to Ethereum’s ERC20 `mapping(address => uint256) public balanceOf`, since both serve as a way to track how many tokens a user owns.

Because a user can have multiple SPL tokens, using just the user’s address as a key would be insufficient to distinguish between balances of different tokens. That’s why the **mint address** is also included in the derivation. By combining both the user’s wallet address and the token’s mint address, Solana ensures that each (user, token) pair gets a **unique** ATA address.

This design avoids collisions and enforces a consistent structure:  

`user_wallet_address + token_mint_address => associated_token_account_address`

To make this clearer, the table below compares how Ethereum and Solana manage token balances.

| Aspect | Ethereum (ERC-20) | Solana (ATAs) |
| --- | --- | --- |
| Storage Model | One central contract stores balances in a mapping | Each user has a separate account (ATA) for each token |
| Balance Location | Stored inside the token contract | Stored inside the user’s ATA |
| Who Pays for Storage | The contract owner (deployment cost) | The user pay for their account |
| Lookup | Call `balanceOf(user)` | Derive ATA address → read balance |
| Parallel Access | Limited by contract | Fully parallel |

Both achieve the same goal—tracking token ownership—but Solana's approach enables parallel processing since each balance is in a separate account.

The diagram below shows the [fields](https://github.com/solana-program/token/blob/main/interface/src/state.rs#L87) of an Associated Token Account.

![a table showing the fields inside the associated token account](https://r2media.rareskills.io/SPLToken/image6.png)

The ATA holds the details of a user's balance of a specific token/mint. Its key fields are:

- `mint`: Address of the token (Mint Account) this account holds. For example, if this is representing a USDC balance, then this would be the address of the USDC mint account
- `owner`:  While this is labeled as ‘owner’, it is the authority of the ATA. The true owner of every ATA is always the Token Program, since it enforces all the rules. The `owner` field here tells the Token Program which authority must sign for updates or transfers to go through. Remember from the [*Owner vs Authority*](https://rareskills.io/post/solana-authority) article, the account’s owner enforces its rule, while the authority is the only valid signer that can send instructions to modify the account, unless signing rights have been delegated through the Token Program by this authority.
- `amount`: The quantity of tokens held in this balance.
- `delegate`: Address of a delegate account, that has been approved to transfer tokens. **Only one delegate can exist at a time for a token account since there is just a single `delegate` field. This differs from ERC-20, where an owner can approve multiple spenders.**
- `state`: The status of the token account, e.g., This is an [enum](https://github.com/solana-program/token/blob/main/interface/src/state.rs#L185) and can be `Uninitialized`, `Initialized` or `Frozen`.
- `close_authority`: Address allowed to close the account, by default the same public key as `owner` but the `owner` can designate another `close_authority`. When a token account's balance reaches zero, the owner can close it to reclaim the SOL used for rent. Several web tools like [Solflare wallet](https://www.solflare.com/) and [Sol-Incinerator](https://sol-incinerator.com/) offer convenient ways to close empty token accounts.

The diagram below shows the relation between the Token Program, the mint account, and token account. 

![a diagram showing the relationship between the token program, mint accounts and token accounts](https://r2media.rareskills.io/SPLToken/image7.png)

**(From this point forward, when we say “token account,” it also includes ATAs, since ATAs are simply a special type of token account.)**

## **Associated Token Account Program**

The Associated Token Account program has the fixed address **`ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`.** It is an onchain program that finds or creates the correct ATA for a given user-token pair. It handles the deterministic address derivation and, when needed, uses [Cross Program Invocation (CPI)](https://www.rareskills.io/post/cross-program-invocation) to create new ATAs through the Token Program.

Specifically, the creation flow orchestrated by the ATA Program is:

![ATA creation flow](https://r2media.rareskills.io/SPLToken/image1.png)

Unlike Ethereum where balances exist implicitly in the contract's storage, Solana requires explicit account creation. This creates a fundamental UX challenge: **tokens cannot be sent to users who haven't explicitly created receiving Associated Token Accounts as the token balance is literally stored in the Associated Token Account.**

Hence, before sending any tokens, we must create the ATA for the user-token pair. The ATA address is derived deterministically off-chain using the wallet address and mint address. Once derived, we use the Associated Token Account Program to create the ATA on-chain if it doesn't already exist. This raises an important security question: if anyone can create an ATA for someone else, could they also assign themselves as the ATA owner and close authority? Fortunately, no. When creating an ATA for another wallet, the ATA Program enforces that the `owner` and `close_authority` fields are always set to the wallet address for which the ATA is being created, not the transaction signer. This security guarantee is built into the [ATA Program's code](https://github.com/solana-program/associated-token-account/blob/c8a2aad5bfc5af81cd95b7685aee30a7ba6f3ba2/program/src/processor.rs#L150), ensuring that only the rightful wallet owner maintains control over their tokens and the ability to close their account.

To illustrate: when Alice wants to send tokens to Bob, she derives Bob's ATA address, creates it on-chain if it doesn't exist, then calls the Token Program's `Transfer` instruction with Bob's ATA as the destination. (In practice, client libraries like `@solana/spl-token` provide helper functions that combine the ATA derivation and creation steps).To illustrate: when Alice wants to send tokens to Bob, she derives Bob's ATA address, creates it on-chain if it doesn't exist, then calls the Token Program's `Transfer` instruction with Bob's ATA as the destination. (In practice, client libraries like `@solana/spl-token` provide helper functions that combine the ATA derivation and creation steps).

The diagram shows the contents of the ATA program. 

![a table showing the instructions for the associated token account program](https://r2media.rareskills.io/SPLToken/image3.png)

We have discussed the accounts involved in creating and managing SPL tokens. Next, we will cover the Token Program and ATA program’s instructions. These instructions let you do things like create & mint new tokens, send tokens between ATAs, set approvals for others to spend your tokens, burn tokens to reduce supply, and close empty accounts to reclaim rent.

## Token Program Instructions

Let's explore the public functions provided by the Token Program that allow you to interact with SPL tokens.

Note that in the instruction parameters below, when we refer to token accounts, these can be either regular token accounts or Associated Token Accounts (ATAs), as discussed earlier in the Token Accounts and Associated Token Accounts section. When a distinction is important, we will explicitly specify whether we're referring to regular token accounts or ATAs.

The Token Program has the following public functions:

**`InitializeMint`**: This instruction creates a new mint account, which represents a new SPL token on-chain.

```rust
pub fn initialize_mint(
    mint_pubkey: &Pubkey,     // The mint account to initialize
    decimals: u8,             // The number of decimal places for the token
    mint_authority: &Pubkey,  // The account with permission to create new tokens
    freeze_authority: Option<&Pubkey> // Optional: The account that can freeze token accounts
) -> Instruction
```

The `mint_pubkey` can be either the address of an unused [keypair account or a PDA](https://rareskills.io/post/solana-pda) that is intended to be initialized as a mint account. We will see this process in practice in the next tutorial.

**`InitializeAccount`**: This instruction initializes a new regular token account (non-ATA) to hold a user’s balance for a specific SPL token mint. 

```rust
pub fn initialize_account(
    account_pubkey: &Pubkey,   // The token account to initialize
    mint_pubkey: &Pubkey,      // The mint for the new token account
    owner_pubkey: &Pubkey      // The owner of the new token account
) -> Instruction
```

ATAs are initialized by the ATA Program, which performs a CPI to this `InitializeAccount` instruction under the hood. We’ll see how later. 

**`Transfer`**: This is used to transfer units of SPL tokens from one user’s token account (source) to another (destination). A “balance” is simply a number stored in the associated token account that only the SPL program can modify. Note that the mint account and token account must be in existence or otherwise created right before the `MintTo` ****instruction is called (also applies for the **`MintTo`** instruction below). We will demonstrate this in the next tutorial using Anchor.

```rust
pub fn transfer(
    source_pubkey: &Pubkey,      // The token account sending tokens (typically the sender's ATA; not the mint account)
    destination_pubkey: &Pubkey, // The destination token account (to which tokens are received)
    authority_pubkey: &Pubkey,   // The owner or delegate authorized to spend from the sending token account
    amount: u64                  // The number of tokens to transfer
) -> Instruction
```

**`MintTo`:** This instruction creates new **token units** and adds them to a specified token account.

```rust
pub fn mint_to(
    mint_pubkey: &Pubkey,        // The token mint address
    account_pubkey: &Pubkey,     // The token account to mint to
    authority_pubkey: &Pubkey,   // The mint's minting authority
    amount: u64                  // The amount to mint
) -> Instruction 
```

**`Burn`**: This instruction destroys a specified amount of SPL token units from a token account, reducing the total token supply. This works similarly to ERC20’s burn function.

```rust
pub fn burn(
    account_pubkey: &Pubkey,     // The token account to burn from
    mint_pubkey: &Pubkey,        // The token mint
    authority_pubkey: &Pubkey,   // The token account's owner/delegate
    amount: u64                  // The amount to burn
) -> Instruction
```

**`Approve`**: This instruction delegates spending power from a token account owner to a specified delegate for a maximum amount. It sets the token account’s `delegate` and an approved amount on that account; only one delegate can exist at a time. Within that limit, the delegate can transfer tokens on behalf of the owner. 

Unlike ERC‑20, which stores allowances in a contract mapping, SPL records the approval directly on the owner’s token account (typically the ATA). This design lets an approval and a transfer be completed in a single transaction, since only the token account’s state is modified.

```rust
pub fn approve(
    source_pubkey: &Pubkey,      // The token account granting approval
    delegate_pubkey: &Pubkey,    // The delegate account
    owner_pubkey: &Pubkey,       // The owner of the token account granting approval
    amount: u64                  // The maximum number of tokens the delegate can transfer
) -> Instruction
```

**`Revoke`**: This instruction cancels any previously granted delegate approval (made with the `Approve` instruction). It removes the delegate entirely by setting the token account’s `delegate` field to `None` (no delegate). 

Since approvals cannot be partially reduced, a new approval with a lesser amount must be set if you want to lower the allowance, similar to how ERC20 decreases allowances.

```rust
pub fn revoke(
    source_pubkey: &Pubkey,      // The token account revoking approval (same account that previously granted approval)
    owner_pubkey: &Pubkey        // The owner of the token account revoking approval
) -> Instruction
```

**`FreezeAccount`**: This instruction is used to freeze a token account, temporarily preventing any transfers or transactions involving the tokens held in the account until it is unfrozen. In other words, SPL supports blacklisting a user’s token account address.

```rust
pub fn freeze_account(
    account_pubkey: &Pubkey,     // The token account to freeze
    mint_pubkey: &Pubkey,        // The token mint
    authority_pubkey: &Pubkey    // The mint's freeze authority
) -> Instruction
```

**`ThawAccount`**: This instruction unfreezes a token account that was previously frozen, allowing token transfers and transactions to resume.

```rust
pub fn thaw_account(
    account_pubkey: &Pubkey,     // The token account to unfreeze
    mint_pubkey: &Pubkey,        // The token mint
    authority_pubkey: &Pubkey    // The mint's freeze authority
) -> Instruction
```

**`SetAuthority`**: This instruction changes who holds certain authority roles on mint and token accounts. 

Recall that a mint account has two fields that have "authority":

- `mint_authority`
- `freeze_authority`

The (associated) token account has two fields that have "authority"

- `owner` the owner of the tokens, NOT the "Solana runtime owner" of the PDA (this naming is confusing).
- `delegate` a public key that can spend tokens on behalf of the owner

The `account_pubkey` in `set_authority` below can refer to either a mint account or a token account.

The `authority_type` specified must match the kind of authority that account holds.

The Solana [source code for SPL](https://github.com/solana-program/token/blob/main/interface/src/instruction.rs#L742) gives an enum name for each of the four kinds of authority:

- `MintTokens`
- `FreezeAccount`
- `AccountOwner`
- `CloseAccount`

Be aware that the SPL program does not refer to authority roles in a consistent manner in the token account and confusingly refers to the token owner as "owner" — this should not be confused with the owner of the PDA.

```rust
pub fn set_authority(
    account_pubkey: &Pubkey,          // The mint or token account
    current_authority_pubkey: &Pubkey, // The current authority
    authority_type: AuthorityType,    // The type of authority to change (e.g., MintTokens, FreezeAccount)
    new_authority_pubkey: Option<&Pubkey> // The new authority, or None to disable
) -> Instruction
```

`Revoke` has an identical effect to `SetAuthority` in that both change who holds an authority. `Revoke` clears the token account’s delegation (sets `delegate` to `None`), while `SetAuthority` changes mint/account authorities (`MintTokens`, `FreezeAccount`, `AccountOwner`, `CloseAccount`).

**`CloseAccount`**: This instruction permanently closes an associated token account and reclaims its SOL lamport balance that was used to make the account rent-exempt. However, the ATA must have exactly zero balance of the underlying mint token, else an error is returned.

```rust
pub fn close_account(
    account_pubkey: &Pubkey,        // The account to close
    destination_pubkey: &Pubkey,    // The account to receive the reclaimed SOL
    owner_pubkey: &Pubkey           // The closing account's owner
) -> Instruction
```

Now let’s discuss the key instructions for the ATA program. 

## Associated Token Account (ATA) Program Instructions

The ATA program works with the Token Program and has these main instructions:

**`Create`:** This instruction creates an ATA at a deterministic PDA address derived from the combination of a wallet address and token mint address. The instruction fails if an account already exists at the derived address.

```rust
pub fn create_associated_token_account(
    payer: &Pubkey,          // The account funding the creation
    wallet_address: &Pubkey, // The wallet address for the ATA
    token_mint: &Pubkey      // The token mint
) -> Instruction
```

**`CreateIdempotent`:** Ensures the correct ATA exists at the derived PDA address. Creates the account if needed. But unlike the `Create` instruction, it succeeds without error even if the correct account already exists.

```rust
pub fn create_associated_token_account_idempotent(
    payer: &Pubkey,          // The account funding the creation
    wallet_address: &Pubkey, // The wallet address for the ATA
    token_mint: &Pubkey      // The token mint
) -> Instruction
```

**Both `Create` and `CreateIdempotent` derive the ATA address and then perform a CPI to the Token Program’s `InitializeAccount`instruction (that we saw earlier) to set up the associated token account.**

## **Summary**

To put it all together, Solana's SPL token architecture is built on a fundamental separation of program logic from token data. Instead of deploying a new contract for every token like on Ethereum, all tokens on Solana are managed by the same core **Token Program**.

Here are the most important points to remember:

- **Logic vs. State:** The single **Token Program** contains all the rules (transfer, mint, burn), but holds no token data (balances, decimals, supply) itself. It acts as the universal logic engine for all SPL tokens.
- **The Mint Account is the Token:** A **Mint Account** defines a unique token. It stores the token’s global information, like its total supply, decimals, and who has the authority to create more. The address of the mint account *is* the token's address (e.g., USDC's mint address).
- **Balances Live in Token Accounts/ATAs:** **User balances are held in separate Token Accounts or Associated Token Accounts (ATAs)**. Rather than one big contract mapping addresses to balances, each user gets a distinct account for each type of token they own. The address for an ATA is predictably derived from the user’s wallet and the token’s mint address, and is the recommended solution.

**Some advantages of the SPL architecture**

- **Parallel Processing:** Because every user's balance is in a separate account, the network can process thousands of transfers at the same time without them getting in each other's way.
- **Standardization:** Every SPL token, whether it's a stablecoin or a meme coin, follows the exact same secure logic from the core Token Program. This reduces the risk of bugs that can happen with custom token contracts.

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*