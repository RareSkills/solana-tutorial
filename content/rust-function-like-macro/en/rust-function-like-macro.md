# Rust function-like procedural Macros

![Rust function-like procedural Macros](https://static.wixstatic.com/media/706568_60a26cb76a6b4396b529e8a4837d50fc~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_60a26cb76a6b4396b529e8a4837d50fc~mv2.jpg)

This tutorial explains the distinction between functions and function like macros. For example, why does `msg!` have an exclamation point after it? This tutorial will explain this syntax.

As a strongly typed language, Rust cannot accept an arbitrary number of arguments to a function.

For example, the Python `print` function can accept an arbitrary number of arguments:

```python
print(1)
print(1, 2)
print(1, 2, 3)
```

The `!` denotes that the "function" is a function-like macro.

Rust function-like macros are identified by the presence of a `!` symbol, for example in `println!(...)` or `msg!(...)` in Solana.

In Rust, a regular function (not function-like macro) to print something is `std::io::stdout().write` and it only accepts a single byte string as an argument.

*If you want to run the following code, the [<ins>Rust Playground</ins>](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021) is a convenient tool if you don’t want to set up a development environment.*

Let’s use the following example (taken from [<ins>here</ins>](https://riptutorial.com/rust/example/1415/console-output-without-macros)):

```rust
use std::io::Write;

fn main() {
    std::io::stdout().write(b"Hello, world!\n").unwrap();
}
```

Note that write is a function, not a macro as it does not have the `!`.

If you try to do what we did above in Python, the code won’t compile because `write` only accepts one argument:

```rust
// this does not compile
use std::io::Write;

fn main() {
    std::io::stdout().write(b"1\n").unwrap();
    std::io::stdout().write(b"1", b"2\n").unwrap();
    std::io::stdout().write(b"1", b"2", b"3\n").unwrap();
}
```

As such, if you wish to print an arbitrary number of arguments, *you need to write a custom print function to handle each case for each number of arguments — that is extremely inefficient!*

Here is what such code would look like (this is highly not recommended!):

```rust
use std::io::Write;

// print one argument
fn print1(arg1: &[u8]) -> () {
		std::io::stdout().write(arg1).unwrap();
}

// print two arguments
fn print2(arg1: &[u8], arg2: &[u8]) -> () {
    let combined_vec = [arg1, b" ", arg2].concat();
    let combined_slice = combined_vec.as_slice();
    std::io::stdout().write(combined_slice).unwrap();
}

// print three arguments
fn print3(arg1: &[u8], arg2: &[u8], arg3: &[u8]) -> () {
    let combined_vec = [arg1, b" ", arg2, b" ", arg3].concat();
    let combined_slice = combined_vec.as_slice();
    std::io::stdout().write(combined_slice).unwrap();
}

fn main() {
    print1(b"1\n");
    print2(b"1", b"2\n");
    print3(b"1", b"2", b"3\n");
}
```

If we look for a pattern in the `print1`, `print2`, `print3` functions, it is simply inserting the arguments into a vector and adding a space in between them, then converting the vector back into a bytes string (a bytes slice to be precise).

Wouldn’t it be nice if we could take a piece of code like `println!` and automatically expand it into a print function that takes exactly as many arguments as we need? 

This is what a Rust macro does.

**A Rust macro takes Rust code as input and programatically expands it into more Rust code.**

This helps us avoid the boredom of having to write a print function for every kind of print statement our code requires.

### Expanding the macro
To see an example of how the Rust compiler is expanding the `println!` macro, check out the [<ins>cargo expand</ins>](https://github.com/dtolnay/cargo-expand) github repo. The result is quite verbose so we will not show it here.

## It’s okay to treat macros as black boxes
Macros are very handy when supplied by a library, but very tedious to write by hand as it requires literally parsing the Rust code.

## Different kinds of macros in Rust
The example we have given with `println!` is a function-like macro. Rust has other kinds of macros but the other two we care about are the *custom derive macro* and the *attribute-like macro*.

Let’s look at a fresh program created by anchor:

![Different kinds of macros](https://static.wixstatic.com/media/935a00_90dbd3b0d418406b8900335888d1516c~mv2.png/v1/fill/w_1480,h_620,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_90dbd3b0d418406b8900335888d1516c~mv2.png)

We will explain how these work in the following tutorial.

## Learn more with RareSkills
This tutorial is part of our free [Solana course](https://www.rareskills.io/solana-tutorial).