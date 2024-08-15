# Introduction to Solana Compute Units and Transaction Fees

![Solana Compute Units](https://static.wixstatic.com/media/935a00_16bc80fd8b6d4b4eb9a0b49fd5ed0cb0~mv2.jpeg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_16bc80fd8b6d4b4eb9a0b49fd5ed0cb0~mv2.jpeg)

In Ethereum, the price of a transaction is computed as $\text{gasUsed} \times \text{gasPrice}$. This tells us how much Ether will be spent to include the transaction on the blockchain. Before a transaction is sent, a gasLimit is specified and paid upfront. If the transaction runs out of gas, it reverts.

Unlike on EVM chains, Solana opcodes/instructions consume "compute units" (arguably a better name) not gas, and each transaction is soft-capped at 200,000 compute units. If the transaction costs more than 200,000 compute units, it reverts.

In Ethereum, gas costs for computing are treated the same as gas costs associated with storage. In Solana, storage is handled differently, so the pricing of persistent data in Solana is a different topic of discussion.

From the perspective of pricing running op codes however, Ethereum and Solana behave similarly.

Both chains execute compiled bytecode and charge a fee for each instruction executed. Ethereum uses EVM bytecode, but Solana runs a modified version of [berkeley packet filter](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) called Solana packet filter.

Ethereum charges different prices for different op codes depending on how long they take to execute, ranging from one gas to thousands of gas. In Solana, each opcode costs one compute unit.

## What to do when you don't have enough compute units
When performing heavy computational operations that cannot be done below the limit, the traditional strategy is to "save your work" and do it in multiple transactions. 

The "save your work" part needs to be put into permanent storage, which is not something we've covered yet. This is similar to if you were trying to iterate over a massive loop in Ethereum; you'd have a storage variable for the index you left off at, and a storage variable saving the computation done up to that point.

## Compute unit optimization
As we already know, Solana uses the compute units to prevent the halting problem and preventing running code that runs forever. It has a compute unit limit per transaction of 200,000 CU (can be increased up to 1.4m CU at some extra cost) which if it (the chosen limit) is exceeded, the program terminates, all changed states revert back and fees are not returned back to the caller. This prevents attackers that intend to run a never ending or computationally intensive program on nodes to slow them down or stop the chain.

However, unlike EVM chains, the computational resources used in a transaction does not affect the fees paid for that transaction. You will be charged as if you used your entire limit or if you used very little of it. For example, a 400 compute unit transaction costs the same as a 200,000 compute unit transaction.

In addition to compute units, the number of [signers for the Solana transaction](https://www.rareskills.io/post/msg-sender-solana) affects the compute unit cost. From the Solana [docs](https://docs.solana.com/developing/intro/transaction_fees#transaction-fee-calculation):

*"So right now, transaction fees are solely determined by the number of signatures that need to be verified in a transaction. The only limit on the number of signatures in a transaction is the max size of transaction itself. Each signature (64 bytes) in a transaction (max 1232 bytes) must reference a unique public key (32 bytes) so a single transaction could contain as many as 12 signatures (not sure why you would do that)."*

We can see this in action with this little example. Start with an empty Solana program like so:
```rust
use anchor_lang::prelude::*;

declare_id!("6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC");

#[program]
pub mod compute_unit {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

### Update the test file:
```javascript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ComputeUnit } from "../target/types/compute_unit";

describe("compute_unit", () => {
  // Configure the client to use the local cluster.
  anchor.setProvider(anchor.AnchorProvider.env());
  const program = anchor.workspace.ComputeUnit as Program<ComputeUnit>;
  const defaultKeyPair = new anchor.web3.PublicKey(
    // replace this with your default provider keypair, you can get it by running `solana address` in your terminal
    "EXJupeVMqDbHk7xY4XP4TVXq22L3ZJxJ9Gm68hJccpLp"
  );

  it("Is initialized!", async () => {
    // log the keypair's initial balance
    let bal_before = await program.provider.connection.getBalance(
      defaultKeyPair
    );
    console.log("before:", bal_before);

    // call the initialize function of our program
    const tx = await program.methods.initialize().rpc();

    // log the keypair's balance after
    let bal_after = await program.provider.connection.getBalance(
      defaultKeyPair
    );
    console.log("after:", bal_after);

    // log the difference
    console.log(
      "diff:",
      BigInt(bal_before.toString()) - BigInt(bal_after.toString())
    );
  });
});
```
*Note: In JavaScript, the "n" at the end of a number means it is a `BigInt`.*

Run: `solana logs` also if you don't already have it running.

When we run `anchor test --skip-local-validator` we get this output as the test logs and Solana validator logs:
```bash
# test logs
		compute_unit
before: 15538436120
after: 15538431120
diff: 5000n


# solana logs
Status: Ok
Log Messages:
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC invoke [1]
  Program log: Instruction: Initialize
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC consumed 320 of 200000 compute units
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC success
```
The balance difference of `5000` lamports is because we need/use only 1 signature (that of our default provider address) when sending this transaction. This is consistent with what we established above, i.e. `1 * 5000 = 5000`. Also notice that this costs 320 in compute units but this amount does not affect our transaction fee.

Now, let's add some complexity to our program and see what happens:
```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let mut a = Vec::new();
    a.push(1);
    a.push(2);
    a.push(3);
    a.push(4);
    a.push(5);

    Ok(())
}
```
Surely, this should make some difference to our transaction fee right?

When we run `anchor test --skip-local-validator` we get this output as the test logs and Solana validator logs:
```bash
# test logs
compute_unit
before: 15538436120
after: 15538431120
diff: 5000n


# solana logs
Status: Ok
Log Messages:
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC invoke [1]
  Program log: Instruction: Initialize
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC consumed 593 of 200000 compute units
  Program 6CCLqLGeyExCFegJDjRDirWQRRSbM5XNq3yKvmaWS2ZC success
```
We can see that this costs more compute units, almost double our first example. But this does not affect our transaction fees. This is expected and shows that truly, compute units do not affect the transaction fees paid by users.

**Regardless of the compute units consumed, the transaction charged 5000 lamports or 0.000005 SOL.**

Back to compute unit. So, why would we want to optimize compute units since it does not affect fees paid for transactions?
+ First, this is only true for now, in the future Solana might conclude to raise the limit to be higher and would have to incentivize nodes to not treat these complex transactions any different than simple ones. This would mean considering compute unit consumed when calculating transaction fees.
+ Second, a smaller transaction is more likely to be included in a block if there is significant network activity competing for blockspace.
+ Third, it will make your program more composable with other programs. If another program calls your program, the transaction does not get extra compute limit. Other programs might not want to integrate with yours if your transaction uses too much compute, leaving little for the original program.

## Smaller integers save compute units
The larger the value types used, the larger the compute unit consumed. It's best to use smaller types where applicable. Let's take the code example and comments:
```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    // this costs 600 CU (type defaults to Vec<i32>)
    let mut a = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);

    // this costs 618 CU
    let mut a: Vec<u64> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);

    // this costs 600 CU (same as the first one but the type was explicitly denoted)
    let mut a: Vec<i32> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);

    // this costs 618 CU (takes the same space as u64)
    let mut a: Vec<i64> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);

    // this costs 459 CU
    let mut a: Vec<u8> = Vec::new();
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);
    a.push(1);

    Ok(())
}
```
Notice the reduction in compute unit cost as the integer type reduced. This is expected as larger types take up larger space in memory than smaller types regardless of the value represented.

Generating a program derived account (PDA) on-chain using `find_program_address` may use more compute units because this method iterates over calls to `create_program_address` until it finds a PDA that's not on the ed25519 curve. To reduce the compute cost, use `find_program_address()` off-chain and pass the resulting bump seed to the program when possible. More on this is discussed in a later section as its out of scope for this section.

This is not an exhaustive list but a few points to give an idea of what makes a program more computationally intensive than another.

## What is eBPF?
Solana's bytecode is heavily derived from BPF. "eBPF" simply means "extended BPF." This section explains BPF in the context of Linux.

Solana VM as you'd expect does not understand Rust or C. Programs written in these languages are compiled down to eBPF (extended Berkeley Packet Filter).

In a nutshell, eBPF allows execution of arbitrary eBPF bytecode within the kernel (in a sandbox environment) when the kernel emits an event the eBPF bytecode subscribes to, example:
+ network: open/close a socket
+ disk: write/read
+ creation of a process
+ creation of a thread
+ CPU instruction invocation
+ supports up to 64 bits (that's why Solana has a max uint type of u64)

You can think of it as JavaScript but for the kernel. JavaScript performs actions on the browser when an event is emitted, eBPF does something very similar for when events are emitted within the kernel, e.g. when a syscall is executed.

This allows us to build programs for various use cases e.g. (based on events listed above):
+ network: To analyze routes and more
+ security: filtering traffic based on certain rules and reporting any bad/blocked traffic
+ tracing and profiling: collecting detailed execution flow from the userspace program to the kernel instructions
+ observability: report and analyze kernel activities

The program is only executed when we need it (i.e. when an event is emitted in the kernel). For example, say you want to get the name of a file and data written to it when it is written to, we listen/register/subscribe to the `vfs_write()` syscall event. Now, whenever that file is written to, we have that data at our disposal.

## Solana Bytecode Format (SBF)
Solana Bytecode Format is a variant of eBPF with certain changes and the one that stands out the most is the removal of the bytecode verifier. The bytecode verifier is present in eBPF to ensure that all possible execution paths are finite and safe to execute.

Solana handles this using compute unit limit . Having a compute meter that limits computational resources spent with a cap moves safety checks to the runtime and allows arbitrary memory access, indirect jumps, loops, and other interesting behaviours.

In a later tutorial, we will deep dive into a simple program and it's bytecode, tweak it, understand the different compute unit costs and learn exactly how Solana bytecode works and how to analyze it.

## Learn more with RareSkills
This tutorial is part of our [Solana course](https://www.rareskills.io/solana-tutorial).

*Originally Published February, 23, 2024*