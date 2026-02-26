# Introduction to sBPF Virtual Machine and Instruction Set

As discussed in the [compute units tutorial](https://rareskills.io/post/solana-compute-unit-price), compute units consumed by a Solana program call equal the number of SBF (Solana Bytecode Format) instructions executed plus the runtime costs of any syscalls. This article dives into the SBF instruction set and demonstrates how to analyze those instructions using execution traces and the `agave-ledger-tool`.

From the Rust to SBF tutorial, we know that Solana programs compile to SBF (Solana Bytecode Format), which runs on the [sBPF virtual machine,](https://github.com/anza-xyz/sbpf) a Solana-specific VM derived from eBPF. SBF instructions resemble traditional assembly languages like x86 or ARM and look like this:

```nasm
mov64 r0, 1  ; move 1 (64 bit padded) to register 0
mov64 r1, 2  ; move 2 (64 bit padded) to register 1
add64 r0, r1 ; add register 1 to register 0, store result in register 0
```

## **Prerequisites**

This article assumes completion of:

- [Compute Units tutorial](https://rareskills.io/post/solana-compute-unit-price) - How Solana compute units work
- Rust to SBF Compilation tutorial - To understand Solana program compilation pipeline
- Basic familiarity with assembly concepts (registers, memory addressing, jumps) is also required.

In the Rust to SBF compilation tutorial, we explained how Solana programs move through three main stages: Rust to LLVM IR to SBF bytecode to native code. This article covers Solana’s VM architecture and shows how to analyze SBF bytecode in practice.

## **The Solana Virtual Machine Architecture**

The Solana VM is register-based, unlike the Ethereum VM, which is stack-based. In a register-based VM, instructions operate on a fixed set of fixed-size storage slots called registers, and each instruction explicitly names which registers it reads from and writes to. In a stack-based VM, instructions implicitly operate on the top of a stack data structure, so operands must be pushed onto the stack and popped off it to be used.

eBPF defines [11 registers](https://docs.kernel.org/6.2/bpf/instruction-set.html#registers-and-calling-convention) (`R0–R10`), all 64 bits wide. Solana's sBPF VM implements these same 11 registers but internally maintains a hidden twelfth register, `R11`, used for program counter tracking. Since `R11` is neither readable nor writable by programs during execution, only the original 11 eBPF registers are visible to program code

From the eBPF specification, the registers have the following use cases:

- `R0`: This register holds the return value from function calls and the program's exit value
- `R1-R5`: These registers hold function call arguments
- `R6-R9`: These are callee-saved registers, meaning they must be preserved across function calls
- `R10`: This is a read-only frame pointer that points to the current stack frame

`R0-R5` are scratch registers that functions can overwrite without saving. `R6-R9` are callee-saved registers that must be preserved across function calls. This means that when a function `foo()` calls `bar()`, any values that `foo()` needs after the call must be kept in `R6–R9`. If `bar()` needs to use those registers, it saves their contents into its own stack frame on entry and restores them before returning to `foo()`. This process is called spilling (saving to stack) and filling (restoring from stack).

**SBF Instruction Set (Opcodes)**

As we know, SBF is based on eBPF, so they use the same instruction set. All instructions (opcodes) that the Solana VM uses are defined [here](https://github.com/anza-xyz/sbpf/blob/main/src/ebpf.rs), and you can also find the complete set with descriptions in the [eBPF specification](https://docs.kernel.org/bpf/standardization/instruction-set.html). 

These opcodes include:

**Arithmetic and Logic Operations:**

- Arithmetic opcodes: `add`, `sub`, `mul`, `div`, `mod`, `neg` (negate),`sdiv` (signed division) and `smod` (signed modulo)
- Logical opcodes: `and`, `or`, `xor`, `lsh` (left shift), `rsh` (right shift), `arsh` ([arithmetic right shift](https://en.wikipedia.org/wiki/Arithmetic_shift))
- Each opcode has a 64-bit variant (the default) and a 32-bit variant
- Each opcode has two forms: one that takes two registers (destination and source), and another that takes one register and an immediate value (a constant hardcoded into the program bytecode). For example, `add64 r0, r1` adds register r1 to r0, while `add64 r0, 42` adds the constant 42 to r0.

**Data Movement:**

- There's also a `mov` opcode that copies values between registers or from immediates to registers

**Control Flow:**

The Solana VM has opcodes for unconditional and conditional jumps.

- `ja` does an unconditional jump. It moves execution to another instruction offset without checking anything.
- Then you have the conditional jumps. `jeq` jumps if two values are equal. `jne` jumps if they are not equal. `jlt` and `jgt` check less than or greater than. `jle` and `jge` check less than or equal, or greater than or equal.
- There are signed versions too. `jslt`, `jsgt`, `jsle`, and `jsge` handle the same comparisons but treat the operands as signed integers.
- `call` moves execution to a labeled part of the bytecode (like a compiled function). For syscalls, programs use the `syscall` instruction (not `call`). The syscall instruction is followed by an identifier specifying which syscall to invoke, for example `sol_log_` or `sol_log_64_`.
- `exit` returns to the caller, or ends program execution if the call stack is empty.

**Memory Operations:**

There are opcodes to perform memory read and write operations too. 

- `ldx` reads from memory into a register, and `stx` writes from a register to memory.
- The load and store instructions have size variants with suffixes that show how many bytes each version reads or writes. So for load, we have: `ldxdw`, `ldxw`, `ldxh`, `ldxb`. So `ldxdw` loads 8 bytes (double word), `ldxw` loads 4 bytes (a word), `ldxh` loads 2 bytes (half word), and `ldxb` loads 1 byte. The store versions follow the same pattern. Since all registers are 64 bits wide, smaller operations (32-bit, 16-bit, 8-bit) write to the lower bits of the register and zero out the upper bits.
- Load takes two operands: a destination register and a memory address. For example, `ldxdw r0, [r1+0x08]` loads 8 bytes from memory at address `r1+0x08` into register `r0`. The syntax `[r1+0x08]` means: take the address stored in `r1`, add a 0x08-byte offset, then read from that final address.
- Store also takes two operands: a memory address and a source register. For example, `stxdw [r1+0x08], r0` stores 8 bytes from register `r0` into memory at address `r1+0x08`.
- Memory addresses are calculated as a base register plus an offset (e.g., `[r1+0x08]`) as seen above.

## **Next Steps**

Now that you understand the sBPF VM architecture, register conventions, and instruction set, the next article demonstrates how to analyze program execution using traces and calculate compute units from actual bytecode execution.

*This article is part of a [tutorial series on Solana](http://rareskills.io/solana-tutorial).*