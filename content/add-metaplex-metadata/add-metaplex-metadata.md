# Implementing Token Metadata with Metaplex

We introduced the Metaplex metadata standard in the previous tutorial. In this one, we’ll create an SPL token and attach metadata to it using the Metaplex standard.

We will build an Anchor program that creates SPL tokens with attached metadata using the Metaplex standard. This allows us to add information to our tokens, such as names, symbols, images, and other attributes.

Before we start building, let's understand the Metaplex standards that govern how token metadata should be structured.

## **Metaplex Token Standards and URI Formats**

When we create metadata for our tokens, we need to follow specific JSON formats defined by Metaplex. The structure depends on the type of token we're creating (NFT, fungible token, etc.).

There are three main standards:

### **Fungible Standard (metadata account `token_standard = 2`)**

This is your regular SPL token with metadata. This is the example we'll create later in this article.

Its Metadata JSON schema is defined like this:

```json
 {
  "name": "Example Token",
  "symbol": "EXT",
  "description": "A basic fungible SPL token with minimal metadata.",
  "image": "https://example.com/images/ext-logo.png"
}

```

### **Fungible Asset Standard (`token_standard = 1`)**

This is analogous to ERC-1155 on Ethereum, used for in-game currency or items. It is defined as a fungible SPL token with a supply greater than 1 but with a zero decimal (i.e. no fractional units).

Its JSON schema includes a few additional fields, such as `attributes`:

```json
{
  "name": "Game Sword",
  "description": "A rare in-game sword used in the battle arena.",
  "image": "https://example.com/images/sword.png",
  "animation_url": "https://example.com/animations/sword-spin.mp4",
  "external_url": "https://game.example.com/item/1234",
  "attributes": [
    { "trait_type": "Damage", "value": "12" },
    { "trait_type": "Durability", "value": "50" }
  ],
  "properties": {
    "files": [
      {
        "uri": "https://example.com/images/sword.png",
        "type": "image/png"
      }
    ],
    "category": "image"
  }
}

```

### **Non-Fungible Standard (`token_standard = 0`)**

This is analogous to ERC-721 on Ethereum — it represents a Non-Fungible Token (NFT). However, on Solana, each NFT is a separate mint with a supply of 1 and 0 decimals, while on Ethereum, ERC-721 uses unique token IDs within a single contract.

The JSON schema for Non-Fungible Standard is identical to the Fungible Asset standard above. Both standards use the exact same metadata structure — the distinction is only on-chain (supply and decimals), not in the JSON format. 

```json
{
  "name": "Rare Art Piece",
  "description": "A one-of-one digital artwork by Artist X.",
  "image": "https://example.com/images/artwork.png",
  "animation_url": "https://example.com/animations/artwork-loop.mp4",
  "external_url": "https://artistx.example.com/rare-art-piece",
  "attributes": [
    { "trait_type": "Artist", "value": "Artist X" },
    { "trait_type": "Year", "value": "2025" }
  ],
  "properties": {
    "files": [
      {
        "uri": "https://example.com/images/artwork.png",
        "type": "image/png"
      }
    ],
    "category": "image"
  }
}
```

Note: On Ethereum, an NFT collection usually lives in a single contract that mints and manages many NFTs. On Solana, each NFT is its own mint, and collections are formed by a verified on-chain link in Metaplex metadata (the collection field we covered in the previous tutorial), not by a single contract. 

Now that we understand these standards, let's build our program to create a fungible token with metadata.

## Implementing a Fungible Standard token

**This is what we'll accomplish:**

1. We’ll create an Anchor program with a function to attach metadata to SPL tokens
2. Create an SPL token (mint account)
3. Use the Metaplex Token Metadata Program via CPI to create a metadata account and link it to the token
4. Store the token URI and its content (like the token image) in a permanent storage, which the token’s metadata account will reference

## Project set up

First, create a new Anchor project with `anchor init spl_with_metadata`.

Then, update the `Anchor.toml` file to this to configure our project properly. We'll set the cluster to 'devnet' since we need to interact with the actual Metaplex Token Metadata Program, which doesn't exist in our local environment. We'll also add the necessary dependencies for working with SPL tokens and metadata:

```toml

[toolchain]
package_manager = "yarn"

[features]
resolution = true
skip-lint = false

[programs.localnet]
spl_token_with_metadata = "ApCjqNHgvuvsiQYpX4kGCxXTipcWJUe7NmnNfq3UKrwD"

[registry]
url = "https://api.apr.dev"

[provider]
cluster = "devnet"                  # added this
wallet = "~/.config/solana/id.json"

[scripts]
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"

```

Next, update the `programs/spl_token_with_metadata/Cargo.toml` file. 

```toml
[package]
name = "spl_token_with_metadata"
version = "0.1.0"
description = "Created with Anchor"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]
name = "spl_token_with_metadata"

[features]
default = []
cpi = ["no-entrypoint"]
no-entrypoint = []
no-idl = []
no-log-ix-name = []
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"] # added "anchor-spl/idl-build"

[dependencies]
anchor-lang = "0.31.0"
anchor-spl = { version = "0.31.0", features = ["token"] } # added this
mpl-token-metadata = "5.1.0"                              # added this
```

We configure our project to use the Anchor SPL and the Metaplex Token Metadata crate. 

We're adding these dependencies for specific purposes:

- `anchor-spl`: Provides Anchor-compatible interfaces for Solana's SPL token program
- `mpl-token-metadata`: Allows us to interact with Metaplex's Token Metadata Program to create and manage metadata for our SPL tokens

We've added the `idl-build = ["anchor-spl/idl-build"]` feature to our Cargo.toml in order to generate an [IDL](https://rareskills.io/post/anchor-idl?utm_source=chatgpt.com) file that includes SPL token types, allowing our TypeScript client to properly interact with our program

## Add the Anchor Program Code

Now update the Anchor program with the following code. 

Here we have defined a `create_token_metadata` function to attach metadata to a provided SPL token. We will explain the code in detail as we proceed further. 

```rust
// Import the necessary dependencies for out program: Anchor, Anchor SPL and Metaplex Token Metadata crate
use anchor_lang::prelude::*;
use anchor_spl::token::Mint;
use mpl_token_metadata::instructions::{
    CreateMetadataAccountV3Cpi, CreateMetadataAccountV3CpiAccounts,
    CreateMetadataAccountV3InstructionArgs,
};
use mpl_token_metadata::types::{Creator, DataV2};
use mpl_token_metadata::ID as METADATA_PROGRAM_ID;

declare_id!("2SZvgGtgotJFy1aKd4Rnm7UEZNxUdP4sdXbeLDgKDiGM"); // run Anchor sync to update your program ID

#[program]
pub mod spl_token_with_metadata {
    use super::*;
    
    pub fn create_token_metadata(
        ctx: Context<CreateTokenMetadata>,
        name: String,
        symbol: String,
        uri: String,
        seller_fee_basis_points: u16,
        is_mutable: bool,
    ) -> Result<()> {
				// Create metadata instruction arguments using the Fungible Standard format
        // This follows the token_standard = 2 format we discussed earlier
        let data = DataV2 {
            name,
            symbol,
            uri, // Points to JSON with name, symbol, description, and image
            seller_fee_basis_points,
            creators: Some(vec![Creator {
                address: ctx.accounts.payer.key(),
                verified: true,
                share: 100,
            }]),
            collection: None,
            uses: None,
        };

        // Find the metadata account address (PDA)
        let mint_key = ctx.accounts.mint.key();
        let seeds = &[
            b"metadata".as_ref(),
            METADATA_PROGRAM_ID.as_ref(),
            mint_key.as_ref(),
        ];
        let (metadata_pda, _) = Pubkey::find_program_address(seeds, &METADATA_PROGRAM_ID);

        // Ensure the provided metadata account matches the PDA
        require!(
            metadata_pda == ctx.accounts.metadata.key(),
            MetaplexError::InvalidMetadataAccount
        );
        
        // Create and execute the CPI to create metadata
        let token_metadata_program_info = ctx.accounts.token_metadata_program.to_account_info();
        let metadata_info = ctx.accounts.metadata.to_account_info();
        let mint_info = ctx.accounts.mint.to_account_info();
        let authority_info = ctx.accounts.authority.to_account_info();
        let payer_info = ctx.accounts.payer.to_account_info();
        let system_program_info = ctx.accounts.system_program.to_account_info();
        let rent_info = ctx.accounts.rent.to_account_info();

        let cpi = CreateMetadataAccountV3Cpi::new(
            &token_metadata_program_info,
            CreateMetadataAccountV3CpiAccounts {
                metadata: &metadata_info,
                mint: &mint_info,
                mint_authority: &authority_info,
                payer: &payer_info,
                update_authority: (&authority_info, true),
                system_program: &system_program_info,
                rent: Some(&rent_info),
            },
            CreateMetadataAccountV3InstructionArgs {
                data,
                is_mutable,
                collection_details: None,
            },
        );
        cpi.invoke()?;
        Ok(())
    }
}

```

We import the necessary dependencies for our program: the SPL token utilities from `anchor_spl` crate for token operations and the Metaplex token metadata components from `mpl_token_metadata` crate for creating and structuring metadata accounts.

The `create_token_metadata` function attaches metadata to an SPL token mint. We'll demonstrate this by calling it with a mint address we create during testing later. 

## Explaining the `create_token_metadata` function

Let’s walk through the `create_token_metadata` function, beginning with the most important part:

### Defining the Metadata Structure

First we define the contents of the metadata account with the `DataV2` struct type which was imported from `mpl_token_metadata::types` (Metaplex Metadata crate).

![A Rust struct holding the metadata for the token](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image10.png)

This structure holds the data for our metadata account and is used by Metaplex when constructing a metadata account. 

### Validating the Metadata Account

Next, we do a check to ensure the right metadata account is passed for the token (mint). 

![a function that gets the metadata PDA](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image1.png)

### Creating the Metadata Account via CPI

Finally, we construct and execute a `CreateMetadataAccountV3` instruction (we discussed this in the previous article) to the Metaplex Token Metadata Program via CPI. 

![A CPI call to create a metadata account](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image2.png)

From the image above, we can see that we pass several accounts to the `CreateMetadataAccountV3Cpi`. These are the accounts required for creating metadata. These accounts passed are defined in the context struct below where we explain each account's source and purpose.

Now, let's look at the context struct for the `create_token_metadata` function, it contains the accounts needed for the `CreateMetadataAccountV3Cpi` CPI call:

The `CreateTokenMetadata` account struct below includes:

- `metadata`: The metadata PDA that will be created by the Metaplex Token Metadata Program
- `mint`: An existing SPL token mint account that we will attach metadata to. We'll create and pass this when testing our program
- `authority`: The mint authority to authorize this transaction
- `payer`: The wallet account that pays for transaction fees and rent
- `system_program`: The Solana System Program for creating new accounts
- `rent`: The Solana Rent sysvar for calculating rent exemption
- `token_metadata_program`: The on-chain Metaplex Token Metadata Program (with fixed address defined as `METADATA_PROGRAM_ID` in our code)

We also define a custom `MetaplexError` error to handle validation failures. Add this code to the program code. 

```rust

#[derive(Accounts)]
pub struct CreateTokenMetadata<'info> {
    /// CHECK: metadata PDA (will be created by the Metaplex Token Metadata program via CPI in the create_token_metadata function)
    #[account(mut)]
    pub metadata: AccountInfo<'info>,

    // The mint account of the token
    #[account(mut)]
    pub mint: Account<'info, Mint>,

    // The mint authority of the token
    pub authority: Signer<'info>,

    // The account paying for the transaction
    #[account(mut)]
    pub payer: Signer<'info>,

    // Onchain programs our code depends on
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,

    /// CHECK: This is the Metaplex Token Metadata program
    #[account(address = METADATA_PROGRAM_ID)]
    // constraint to ensure the right account is passed
    pub token_metadata_program: AccountInfo<'info>,
}

#[error_code]
pub enum MetaplexError {
    #[msg("The provided metadata account does not match the PDA for this mint")]
    InvalidMetadataAccount,
}

```

Let’s implement this test for our program. 

## Testing our program

### Understanding Irys

[Irys](https://irys.xyz/) (formerly Bundlr) is a service that makes it easy to upload data to [Arweave](https://arweave.org/), a permanent decentralized storage. In our test, we’ll use Irys to upload both our token's image and metadata JSON to permanent storage.

**Key points about Irys:**

- It’s already included in the `@metaplex-foundation/js` package through the `irysStorage` module, so no additional dependency installation is required after installing `@metaplex-foundation/js`. You can simply import it.
- No separate account creation is required as it uses our existing Solana wallet for payments
- In devnet, uploads are paid for using devnet SOL from your wallet (we will request for devnet SOL via airdrop later)

Now update the program test in `tests/spl_token_with_metadata.ts` with the code below.

The test does the following:

1. It defines the Metaplex Token Metadata Program ID as constant (needed for PDA derivation and CPI calls)
2. Creates an SPL token using a generated keypair account
3. Sets up Metaplex with Irys to upload a locally stored image to the Arweave devnet, which returns a URI to the uploaded image
4. Creates JSON metadata in the test (with name, symbol, description, and the image URI) and uploads it to Arweave
5. Derives the metadata PDA using 'metadata' prefix, metadata program ID, and mint address as seeds
6. Calls `create_token_metadata` in our Anchor program
7. Finally, it verifies the metadata account exists and has the correct owner

There are added code comments to show each step.

```tsx

import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import {
  Metaplex,
  irysStorage,
  keypairIdentity,
  toMetaplexFile,
} from "@metaplex-foundation/js";
import { createMint } from "@solana/spl-token";
import { Keypair, PublicKey, SystemProgram } from "@solana/web3.js";
import { assert } from "chai";
import { readFileSync } from "fs";
import path from "path";
import { SplTokenWithMetadata } from "../target/types/spl_token_with_metadata";

describe("spl_token_with_metadata", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const program = anchor.workspace
    .splTokenWithMetadata as Program<SplTokenWithMetadata>;
  const wallet = provider.wallet as anchor.Wallet;

  const TOKEN_METADATA_PROGRAM_ID = new PublicKey(
    "metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s"
  );

  // configure Metaplex with Irys (formerly Bundlr)
  const metaplex = Metaplex.make(provider.connection)
    .use(keypairIdentity(wallet.payer))
    .use(
      irysStorage({
        address: "https://devnet.irys.xyz", // Irys endpoint
        providerUrl: provider.connection.rpcEndpoint,
        timeout: 60_000,
      })
    );

  it("creates token with metadata", async () => {
    // Create the mint
    const mintKeypair = Keypair.generate();
    await createMint(
      provider.connection,
      wallet.payer,
      wallet.publicKey,
      wallet.publicKey,
      9,
      mintKeypair
    );
    const mintPubkey = mintKeypair.publicKey;
    console.log("Mint Pubkey:", mintPubkey.toBase58());

    // Read & convert our image into a MetaplexFile
    const imageBuffer = readFileSync(
      path.resolve(__dirname, "../assets/image/kitten.png")
    );
    const metaplexFile = toMetaplexFile(imageBuffer, "kitten.png");

    // Upload image, get arweave URI string
    const arweaveImageUri: string = await metaplex.storage().upload(metaplexFile);
    const imageTxId = arweaveImageUri.split("/").pop()!;
    const imageUri = `https://devnet.irys.xyz/${imageTxId}`;
    console.log("Devnet Irys image URL:", imageUri); // using Irys devnet gateway because Arweave public gateway has no devnet

    // Build our JSON metadata object following the Fungible Standard format
    // This matches the token_standard = 2 format we explained earlier
    const metadata = {
      name: "Test Token",
      symbol: "TEST",
      description: "Test token with metadata example",
      image: imageUri,
    };

    // Upload JSON, get arweave URI string
    const arweaveMetadataUri: string = await metaplex
      .storage()
      .uploadJson(metadata);
    const metadataTxId = arweaveMetadataUri.split("/").pop()!;
    const metadataUri = `https://devnet.irys.xyz/${metadataTxId}`;
    console.log("Devnet Irys metadata URL:", metadataUri); // using Irys devnet gateway because Arweave public gateway has no devnet

    // Derive on-chain metadata PDA
    const [metadataPda] = PublicKey.findProgramAddressSync(
      [
        Buffer.from("metadata"),
        TOKEN_METADATA_PROGRAM_ID.toBuffer(),
        mintPubkey.toBuffer(),
      ],
      TOKEN_METADATA_PROGRAM_ID
    );
    console.log("Metadata PDA:", metadataPda.toBase58());

    // Call the create_token_metadata function
    const tx = await program.methods
      .createTokenMetadata(
        metadata.name,
        metadata.symbol,
        metadataUri,
        100, // 1%
        true // isMutable
      )
      .accounts({
        metadata: metadataPda,
        mint: mintPubkey,
        authority: wallet.publicKey,
        payer: wallet.publicKey,
        systemProgram: SystemProgram.programId,
        rent: anchor.web3.SYSVAR_RENT_PUBKEY,
        tokenMetadataProgram: TOKEN_METADATA_PROGRAM_ID,
      })
      .rpc();
    console.log("Transaction signature:", tx);

    // Assert the account exists & is owned by the Metadata program
    const info = await provider.connection.getAccountInfo(metadataPda);
    assert(info !== null, "Metadata account must exist");
    assert(
      info.owner.equals(TOKEN_METADATA_PROGRAM_ID),
      "Wrong owner for metadata account"
    );
  });
});

```

Now that we have our test ready, create an `assets/image` directory in our workspace to place the image we will use as our token image. This image will be used by Irys in our test at:

![Typescript code to upload images to Irys](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image3.png)

We have already uploaded this image to Irys devnet for you. Click the link below to download it and place it in the directory you just created: [https://devnet.irys.xyz/8VY89xG1RiUjtz1Lwgip7eUxZvtsdkf1gViGYaDKmwx8](https://devnet.irys.xyz/8VY89xG1RiUjtz1Lwgip7eUxZvtsdkf1gViGYaDKmwx8)

Now, run `npm install @solana/spl-token @metaplex-foundation/js` on the terminal to install the dependencies for the test.

Next, configure Solana to use devnet. 

Run this command on the terminal: `solana config set --url [https://api.devnet.solana.com](https://api.devnet.solana.com/)`

![A shell command to switch the default network to testnet](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image4.png)

Request for airdrop: `solana airdrop 5`. We need funds to deploy our program and all accounts involved to devnet.  

![Airdropping 5 sol on the testnet](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image5.png)

Now build the project and run the test. This will deploy the program, token and metadata account to the Solana devnet (note: this might take some time depending on your network connection). 

![A test showing the successful creating of a token with metadata](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image6.png)

We can see the mint (token) has been created with metadata (and has an image), as shown below. 

![Test token on the block explorer with metadata visible](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image7.png)

We can also view its metadata as well. 

![A JSON holding the metadata of the token](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image8.png)

Deployment link for this particular one: [https://explorer.solana.com/tx/2c27FRN48fHzzLTA9kV2XXwCEUPEQcWXvfT3k31PhPoEyFNe3bepJ7XxvKwAXekzPaV5nQeCR8mfxAeKqG15QT4Q?cluster=devnet](https://explorer.solana.com/tx/2c27FRN48fHzzLTA9kV2XXwCEUPEQcWXvfT3k31PhPoEyFNe3bepJ7XxvKwAXekzPaV5nQeCR8mfxAeKqG15QT4Q?cluster=devnet)

Metadata account: [https://explorer.solana.com/address/5feQdhNd3PxPJ9apKUpCWfB47cQdLitNMrVP8Gnq3cad?cluster=devnet](https://explorer.solana.com/address/5feQdhNd3PxPJ9apKUpCWfB47cQdLitNMrVP8Gnq3cad?cluster=devnet)

To see the effect of the metadata account, look at the UI for the deployment of a regular SPL token (which we did in previous tutorials). Notice how there is no name or image at the top, and no tab for Metadata next to the History, Transfer, and Instructions tabs.

![A screenshot of a token with metadata missing](https://r2media.rareskills.io/SolanaImplementTokenMetadataMetaplex/image9.png)

We have successfully deployed an SPL token and attached metadata to it using the Metaplex standard, following the **Fungible Standard** format we discussed earlier with the basic name, symbol, description, and image fields.

## Conclusion

In this tutorial, we created an SPL token and attached metadata to it with the help of the Metaplex Token Metadata standard. 

In our Anchor program, we used the `DataV2` struct to define the token’s metadata and invoked the `CreateMetadataAccountV3` instruction (through CPI) from the `mpl-token-metadata` crate to create the metadata account. We used Metaplex with Irys to upload the token's image and metadata JSON to Arweave. We then confirmed that the metadata account exists after creation and is owned by the Metaplex program. Lastly, we explained the Metaplex token standards — Fungible (2), Fungible Asset (1), and Non-Fungible (0) — and outlined their JSON URI formats.

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*