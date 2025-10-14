# How Metaplex Metadata for Tokens Works

We have deployed and interacted with [SPL tokens](https://rareskills.io/post/spl-token), but none of them had a name, symbol, or any metadata attached. Instead, we identified each token by its mint account address. In contrast, ERC20 tokens include functions for reading the token’s name and symbol (but it's worth noting these are just human-readable conveniences, there is nothing preventing different tokens from having the same name or symbol). [ERC721](https://rareskills.io/post/erc721) and [ERC1155](https://rareskills.io/post/erc-1155) also include a `tokenURI` function that returns a URI pointing to off-chain metadata. 

But as we have seen so far, the SPL mint account does not have a name, symbol, or a URI field.

There are two main solutions for this on Solana:

- The Metaplex Token Metadata Standard
- The SPL Token-2022

**The Metaplex Token Metadata Standard**: This is the most widely used approach for adding metadata to tokens on Solana. When you see NFTs with images or tokens with names and symbols (like the famous "dog wif hat" memecoin), they're likely using this standard. It works through a separate metadata account that is linked to your token. This is the opposite of how ERC-721 works - in ERC-721, the token contract points to the metadata, but in Metaplex, the metadata account points to the token mint.

![This image shows the relationship between an SPL Token Mint, Metadata Account, and Off-chain JSON for additional metadata](https://r2media.rareskills.io/SolanaMetaplexTokenMetadata/image3.png)

This image shows the relationship between an SPL Token Mint, Metadata Account, and Off-chain JSON for additional metadata

**The SPL Token-2022:** This is a separate program from the original SPL Token program that includes built-in support for token metadata and other advanced features. Although more modern, it is not yet as widely adopted as the Metaplex standard at the time of writing.

We will focus on the Metaplex solution today and cover Token-2022 in a separate article.

We'll break our coverage of the Metaplex solution into two parts: In this article, we'll introduce Metaplex and discuss how it handles metadata for SPL tokens. In the next one, we'll implement Metaplex metadata for an SPL token using Anchor. 

## What is Metaplex and how does it provide token metadata?

Metaplex is a set of open standards and tools built on Solana. These standards are maintained by the Metaplex Foundation and have become the primary way to create and manage digital assets on Solana.

**In practical terms:**

- When you see an NFT or token with an image in your Phantom wallet, that image link likely comes from Metaplex metadata
- When a token has a name and symbol in your wallet instead of just an address, it is most likely using Metaplex metadata
- Popular NFT collections on Solana (like DeGods or Okay Bears) use Metaplex standards

This is what it looks like: [https://explorer.solana.com/address/EKpQGSJtjMFqKZ9KQanSqYXRcF8fBopzLHYxdM65zcjm/metadata](https://explorer.solana.com/address/EKpQGSJtjMFqKZ9KQanSqYXRcF8fBopzLHYxdM65zcjm/metadata)

![screenshot of token metadata on the Solana block explorer](https://r2media.rareskills.io/SolanaMetaplexTokenMetadata/image1.png)

Metaplex provides several tools, but our focus for now is the **Token Metadata Program**, which lets you attach metadata to any SPL token.

### Metaplex Token Metadata Program

The Metaplex Token Metadata Program, with the address [`metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`](https://solscan.io/account/metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s), is used to add extra metadata such as name, symbol, image, and description to SPL tokens. It creates and manages **metadata accounts**, which store structured metadata tied to a specific SPL mint.

Like the SPL Token Program owns all mint accounts and ATAs, the Metaplex Token Metadata Program owns all metadata accounts under it. It is an executable and upgradeable program.

At a high level, the program lets us:

1. Create a metadata account for an SPL token mint to store structured data like name, symbol, and URI in this account
2. Update this data with specific instructions

We will go into more detail about the instructions in the "Metaplex Token Metadata Program Instructions" section below.

Now let's look at how these metadata accounts are structured and created.

### Metaplex Metadata Account

A Metaplex metadata account is a Program Derived Address (PDA) created through the Metaplex Token Metadata Program to attach extra data to an SPL token. The PDA is derived using three seeds: the string `"metadata"`, the `token_metadata_program_id`, and the `mint_account_address`.

Only the mint address varies; the `"metadata"` string and program ID are fixed. This ensures only one metadata account can be derived for a given mint account. 

While the metadata account stores basic on-chain information about the token, it can also hold a URI that points to off-chain resources. For example, an NFT's metadata account might store the token name and symbol on-chain, while the URI points to a JSON file containing the full description, image, and additional attributes stored on IPFS, Arweave or a web server.

The diagram below shows the relation between the Metaplex Token Metadata Program and the Metadata account. 

![a diagram showing the relationship of the token metadata account and the mint account](https://r2media.rareskills.io/SolanaMetaplexTokenMetadata/image4.png)

The image below shows the the fields relevant to our discussion of a metadata account, don’t worry if they don’t all make sense yet, we will explain them later. 

![a table showing the metadata account fields](https://r2media.rareskills.io/SolanaMetaplexTokenMetadata/image2.png)

The metadata account stores the metadata (name, symbol, image, etc.) about a mint account. Since the address of the metadata account can be directly derived from the address of the mint account, wallets can easily discover the metadata account if it exists.

From the image above, here’s what each key represents:

- `key`: This is an account discriminator (currently **MetadataV1**). While the variable name `key` isn't the most descriptive (a more accurate name would be `version` or `discriminator`), this is what Metaplex uses in their codebase. It indicates the metadata structure version currently being used and lets wallets and tools parse the account correctly. Each mint can only have one metadata account because of the PDA derivation mechanism which uses the mint address as the only seed that does not have a fixed value.
- `update_authority`: The wallet address with the authority to update the metadata account. This update authority is set during the metadata account creation, and only this authority can modify the metadata afterward. The update authority can transfer these rights to another address or permanently give up control by setting the authority to null, making the metadata immutable. Every update operation requires a valid signature from the current authority address, and this prevents unauthorized changes to the token's metadata. We'll discuss the specific instructions for updating and transferring authority later in this article.
- `mint`: This points to the SPL token mint account described by this metadata. This links the metadata to the actual token.
- `data`: This field represents the SPL token asset data. It contains the following:
    - `name`: The name of the token (e.g., "Circle US Dollar Stablecoin"). Limited to 32 bytes, so names must be concise. As defined [here](https://github.com/metaplex-foundation/mpl-token-metadata/blob/23aee718e723578ee5df411f045184e0ac9a9e63/programs/token-metadata/program/src/state/metadata.rs#L16).
    - `symbol`: A short identifier for the token (e.g., "USDC"). Limited to 10 bytes, and typically in uppercase.
    - `uri`: A link to an off-chain JSON file that contains the extended metadata, similar to how NFT URIs work in ERC721 and ERC1155. This JSON adheres to a standardized format, and includes fields for description, image, animation URL, attributes, and more. The URI has a size limit of 200 bytes. We will delve into this JSON in more detail later
    - `seller_fee_basis_points`: This represents the royalty fee to be applied to secondary sales of the token, expressed in basis points (where 1% equals 100 basis points). Marketplaces use this information to determine the amount they should remit to creators upon selling the token. The royalty information is a suggestion for marketplaces rather than an enforced rule, and the permissible range for these values is from 0 to 10,000 (i.e., 0% to 100%).
    - `creators`: A vector of up to [5 addresses](https://github.com/metaplex-foundation/mpl-token-metadata/blob/23aee718e723578ee5df411f045184e0ac9a9e63/clients/rust/src/lib.rs#L19) that serves as the creator(s) of the token, with each address assigned a percentage of the seller fee (royalties). The total of these percentages must equal 100. This use of the `creators` field is a recommendation for marketplaces, not a strict rule.
    - In addition, `data` can include an optional `collection` field for NFTs. This field holds two pieces of information: **a public key** and **a boolean flag**. The public key is the mint address of another NFT that serves as the collection's identifier. Unlike Ethereum where collections are typically contract-based, Metaplex uses a special NFT to represent an entire collection. This "collection NFT" uses the same metadata account as regular NFTs, but its metadata account includes an additional `collection_details` field that identifies it as a collection. The boolean flag (the second piece of information) is set by the collection's update authority to verify that the NFT legitimately belongs to that collection. This on-chain verification system allows wallets and marketplaces to group and display NFTs by their collection.
- `primary_sale_happened`: This is a boolean flag that tracks if the initial sale occurred. Once set to true, it cannot be reversed. This affects royalty distributions in some marketplaces, because most marketplaces start enforcing royalties only after the primary sale has occurred.
- `is_mutable`: Determines if the metadata can be updated after creation. If set to false, the metadata becomes permanently frozen (meaning no further updates are possible, even by the update authority).
- `edition_nonce`**:** An optional field used for Master Editions, an advanced feature that allows creating limited edition copies of NFTs. This is not essential for understanding how metadata works. If you're interested in this feature, you can learn more in the [Metaplex documentation](https://developers.metaplex.com/token-metadata/print).
- `token_standard`: An enum that identifies the token type, it can be any of the following:
    - `Fungible` (standard SPL tokens),
    - `NonFungible` (unique NFTs with no editions),
    - `FungibleAsset` (fungible tokens with metadata, it can be in-game items with similar use-case with ERC1155)

## Metaplex Token Metadata Program Instructions

The Metaplex Token Metadata Program has several instructions we can call from a program or a client. With them, we create and manage metadata for SPL tokens. 

The instructions are:

- `CreateMetadataAccountV3`: This instruction creates a new metadata account for an SPL token mint. It attaches metadata like name, symbol, URI, and creator information to the token. This is the primary way to give your token an identity beyond just its mint address. `V3` here just means it is the third version of this instruction. Only the mint authority of the token can create metadata for it, this prevents anyone from attaching unauthorized metadata to tokens they don't control.
- `UpdateMetadataAccountV2`: This instruction updates the metadata of an existing token. It can modify fields like name, symbol, URI, creators, transfer authority to a new address, and other attributes as long as the metadata is marked as mutable and the update authority signs the transaction. Only the designated update authority can modify the metadata, this authority is set when the metadata is first created.
- `UpdatePrimarySaleHappenedViaToken`: This instruction marks that the primary sale of an NFT has happened. Once set to true, it cannot be reversed. This affects royalty distributions in some marketplaces, as most start enforcing royalties only after the primary sale. Only the update authority can invoke this instruction.
- `SignMetadata`: This instruction allows creators listed in the metadata to verify their identity by signing the metadata. When a creator signs, their "verified" status becomes true, which helps buyers confirm the authenticity of the token.
- `CreateMasterEditionV3`: This instruction creates a master edition account for an NFT. It can only be called once per mint, and only if the supply is exactly 1. It transforms a regular NFT into a master edition that can produce limited edition copies. As mentioned earlier, only the update authority for the mint can invoke this instruction.
- `MintNewEditionFromMasterEditionViaToken`: This instruction creates a new limited edition NFT from a master edition. The new edition has its own mint and metadata but is linked to the master edition.
- `SetAndVerifyCollection`: This instruction links an NFT to a collection and verifies it in a single step. It requires the collection's update authority to sign the transaction, confirming that the NFT is an official part of the collection.
- `VerifySizedCollectionItem`: This instruction verifies that an NFT belongs to a sized collection. Sized collections track the exact number of items they contain, which helps with accurate representation in marketplaces.
- `VerifyCollection`: This instruction verifies that an NFT belongs to a collection. It requires the collection's update authority to sign, confirming the NFT's membership in the collection.
- `UnverifyCollection`: This instruction removes the verification status of an NFT from a collection. It requires either the collection's update authority or the NFT's update authority to sign.
- `Utilize`: This instruction tracks the usage of an NFT that has the "uses" feature enabled. This allows NFTs to have a limited number of uses before they expire or change state.
- `ApproveUseAuthority`: This instruction delegates the authority to utilize an NFT to another address, allowing someone else to use the NFT a specified number of times.
- `RevokeUseAuthority`: This instruction revokes a previously granted use authority, preventing the delegate from further using the NFT.
- `ApproveCollectionAuthority`: This instruction delegates the authority to verify collection items to another address.
- `RevokeCollectionAuthority`: This instruction revokes a previously delegated collection authority.
- `BurnNft`: This instruction burns an NFT, destroying both the token and its metadata. This is irreversible and removes the NFT from circulation.

## Less Commonly Used Metaplex Token Metadata Instructions

The following instructions are less commonly used but still part of the Metaplex Token Metadata Program:

- `CreateMetadataAccount`: Legacy version of `CreateMetadataAccountV3`.
- `CreateMetadataAccountV2`: Legacy version of `CreateMetadataAccountV3`.
- `UpdateMetadataAccount`: Legacy version of `UpdateMetadataAccountV2`.
- `CreateMasterEdition`: Legacy version of `CreateMasterEditionV3`.
- `CreateMasterEditionV2`: Legacy version of `CreateMasterEditionV3`.
- `VerifySizedCollectionItem`: This verifies an item as part of an NFT collection.
- `SetAndVerifySizedCollectionItem`: This sets and verifies an item as part of an NFT collection in one instruction.
- `FreezeDelegatedAccount`: Freezes a delegated account.
- `ThawDelegatedAccount`: Thaws (unfreezes) a previously frozen delegated account.
- `RemoveCreatorVerification`: Removes a creator's verification from an NFT.
- `PuffMetadata`: Extends the metadata account to its maximum size by padding shorter fields with null bytes. When metadata fields (like name, symbol, URI) are shorter than their maximum allowed length, this instruction fills the remaining space with zeros. The [maximum sizes are: name (32 bytes), symbol (10 bytes), and URI (200 bytes)](https://github.com/metaplex-foundation/mpl-token-metadata/blob/a7ee5e17ed60feaafeaa5582a4f46d9317c1b412/programs/token-metadata/program/src/state/metadata.rs#L16-L20).

### Conclusion

To wrap up: the Metaplex Token Metadata Program gives SPL tokens an identity like name, symbol, and image, through a separate metadata account tied to the mint. This account is a PDA and stores structured fields, including a URI that points to an off-chain JSON. We broke down what each field means and how it affects how wallets and marketplaces handle the token. Finally, we went through the instructions the program provides and what they each do. 

In the next tutorial, we will create metadata for an SPL token. 

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*
