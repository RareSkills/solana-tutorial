# Solana programs are upgradeable and do not have constructors

![Hero image showing Anchor deploy](https://static.wixstatic.com/media/935a00_6f744496166444cbbd0621def8ead449~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_6f744496166444cbbd0621def8ead449~mv2.jpg)

In this tutorial we will peek behind the scenes of anchor to see how a Solana program gets deployed.

Let’s look at the test file anchor creates for us when we run `anchor init deploy_tutorial`:

```javascript
describe("deploy_tutorial", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.DeployTutorial as Program<DeployTutorial>;

  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods.initialize().rpc();
    console.log("Your transaction signature", tx);
  });
});
```

The starter program it generates should be familiar:

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod deploy_tutorial {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

Where and when is the program above deployed?

The only plausible place the contract could have been deployed is on the line in the test file:

```javascript
const program = anchor.workspace.DeployTutorial as Program<DeployTutorial>;
```

But that doesn’t make sense, because we would expect that to be an async function.

Anchor is silently deploying the program in the background.

## Solana programs do not have constructors
This may seem unusual to those coming from other object-oriented languages. Rust does not have objects or classes.

In an Ethereum smart contract, a constructor can configure storage or set bytecode and immutable variables.

So where exactly is the “deployment step”?

*(If you are still running the Solana validator and Solana logs, we recommend you restart and clear both terminals)*

Let’s do the usual setup. Create a new Anchor project called program-deploy, and make sure the validator and logs are running in other shells.

Instead of running `anchor test`, run the following command in the terminal:

```bash
anchor deploy
```

![Screenshot for Anchor test](https://static.wixstatic.com/media/935a00_fb4d75f1016144a3bc2e2706c27736f0~mv2.png/v1/fill/w_1480,h_424,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_fb4d75f1016144a3bc2e2706c27736f0~mv2.png)

In the screenshot of the logs above, we can see the point at which the program was deployed.

Now here’s the interesting part. Run `anchor deploy` again:

![Screenshot-upgraded for Anchor test](https://static.wixstatic.com/media/935a00_95bc7ab0a8de400aac1f11e47471b748~mv2.png/v1/fill/w_1480,h_296,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_95bc7ab0a8de400aac1f11e47471b748~mv2.png)

The program was deployed to the same address, but this time it was *upgraded*, not deployed.

The program id has not changed, **the program got overwritten**.

## Solana programs are mutable by default
This might come as a shock to Ethereum developers where immutability is assumed. 

What is the point of a program if the author can just change it? It is possible to make a Solana program immutable. The assumption is that the author will deploy the mutable version first, and after time goes by and no bugs are discovered, then they will redeploy it as immutable.

Functionally, this is no different than an administrator controlled proxy where the owner later forfeits ownership to the zero address. But arguably, the Solana model is a lot cleaner because a lot can go wrong with Ethereum proxies.

Another implication: **Solana does not have delegatecall, because it doesn’t need it**. 

The primary use of delegatecall in Solidity contracts is to be able to upgrade the functionality of a proxy contract by issuing delegatecalls to a new implementation contract. However, since the bytecode of a program in Solana can be upgraded, there is no need for delegatecalling to implementation contracts.

Yet another corollary: **Solana does not have immutable variables the way Solidity interprets them (variables that can only be set in the constructor)**.

## Running tests without redeploying the program
Since anchor by default redeploys the program, let's demonstrate how to run the tests without redeploying.

Alter the test to be the following:

```javascript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";

import fs from 'fs'
let idl = JSON.parse(fs.readFileSync('target/idl/deployed.json', 'utf-8'))

describe("deployed", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());

  // Change this to be your programID
  const programID = "6p29sM4hEK8ZFT5AvsGJQG5nKUtHBKs13iVL6juo5Uqj";
  const program = new Program(idl, programID, anchor.getProvider());

  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods.initialize().rpc();
    console.log("Your transaction signature", tx);
  });
});
```

Before you run the test, we suggest clearing the terminal of the Solana logs and restarting the solana-test-validator.

Now, run the test with:

```bash
anchor test --skip-local-validator --skip-deploy
```

Now look at the logs terminal:


![Screenshot for Anchor test when skipping deploy](https://static.wixstatic.com/media/935a00_177b1f145f08486d94416f73f502c14e~mv2.png/v1/fill/w_1480,h_388,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_177b1f145f08486d94416f73f502c14e~mv2.png)

We see that the initialize instruction was executed, but the program was neither deployed nor upgraded, since we used the `--skip-deploy` argument with anchor test.

**Exercise**: To see that the program bytecode actually changed, deploy two contracts that print different `msg!` values. Specifically,
1. Update the `initialize` function in [lib.rs](https://lib.rs/) to include a `msg!` statement that writes a string to the logs.
2. `anchor deploy`
3. `anchor test --skip-local-validator --skip-deploy`
4. Check log to see the message logged
5. Repeat 1 - 4, but change the string in the `msg!`
6. Verify the program id did not change

You should observe message string changes but the program id stays the same.

## Summary
- Solana does not have a constructor, programs are “just deployed”
- Solana does not have immutable variables
- Solana does not have delegatecall, programs can be “just updated”

## Learn more with RareSkills
This tutorial is part of our free [Solana course](https://www.rareskills.io/solana-tutorial).

*Originally Published February, 12, 2024*