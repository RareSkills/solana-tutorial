# Visibility and "inheritance" in Rust and Solana

![Rust function visibillity](https://static.wixstatic.com/media/935a00_0cc717e55a18467587b68a28852aa476~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0cc717e55a18467587b68a28852aa476~mv2.jpg)

Today we will be learning how Solidity’s function visibility and contract inheritance can be conceptualized in Solana. There are four levels of function visibility in Solidity, they are:

- public - accessible from within the contract and externally.
- external - accessible from outside the contract only.
- internal - accessible within the contract and inheriting contracts.
- private - accessible within the contract only.

Let’s achieve the same in **Solana**, shall we?

## Public functions
All the functions we have defined since day1 till date are all public functions:

```rust
pub fn my_public_function(ctx: Context<Initialize>) -> Result<()> {
    // Function logic...

    Ok(())
}
```

Adding `pub` keyword prior to function declaration makes the function public.

You cannot remove the `pub` keyword for functions inside of the module (mod) labeled `#[program]`. It will not compile.

### Don’t worry too much about the distinction between external and public
It’s generally inconvenient for a Solana program to call it’s own public function. If there is a `pub` function in a Solana program, for all practical purposes you can think of it as external in the context of Solidity.

If you want to call a public function inside the same Solana program, it's much easier to wrap the public function with an internal implementation function and call that.

## Private and Internal Functions
Although you cannot declare functions without `pub` inside the module with the `#[program]` macro, you can declare functions inside the file. Consider the following code:

```rust
use anchor_lang::prelude::*;

declare_id!("F26bvRaY1ut3TD1NhrXMsKHpssxF2PAUQ7SjZtnrLkaM");

#[program]
pub mod func_test {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
				// -------- calling a "private" function --------
        let u = get_a_num();
        msg!("{}", u);
        Ok(())
    }
}

// ------- We declared a non pub function over here -------
fn get_a_num() -> u64 {
    2
}

#[derive(Accounts)]
pub struct Initialize {}
```

This will run and log as expected.

This is all you really need to know about public and internal functions if you want to build simple Solana programs, but if you want to organize your code better than just declaring a bunch of functions in the file outside the program, you can keep going. **Rust, and hence Solana, does not have “classes” the way Solidity does, as Rust is not object-oriented. Hence, the distinction between “private” and “internal” is doesn’t have a straightforward analog to Rust.**

Rust uses modules to organize code. The visibility of functions inside and outside these modules is well discussed in the [<ins>Visibility and Privacy section of the Rust docs</ins>](https://doc.rust-lang.org/beta/reference/visibility-and-privacy.html), but we will add our own Solana-oriented take below.

## Internal function
This can be achieved by defining the function within the program module and making sure it is accessible within its own module and other modules where it is imported or used. Let’s see how to do that:

```rust
use anchor_lang::prelude::*;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        // Call the internal_function from within its parent module
        some_internal_function::internal_function();

        Ok(())
    }

    pub mod some_internal_function {
        pub fn internal_function() {
            // Internal function logic...
        }
    }
}

mod do_something {
    // Import func_visibility module
    use crate::func_visibility;

    pub fn some_func_here() {
        // Call the internal_function from outside its parent module
        func_visibility::some_internal_function::internal_function();

        // Do something else...
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

After building the program, if you navigate to the `./target/idl/func_visibility.json` file, you will observe that the function defined within the `some_internal_function` module was not included in the built program. This indicates that the function `some_internal_function` is internal and can only be accessed within the program itself and any programs that import or use it.

From the example above, we were able to access `internal_function` function from within its "parent" module (`func_visibility`) and also from a separate module (`do_something`) outside the `func_visibility` module.

## Private function
Defining a function within a specific module and ensuring they are not exposed outside that scope is a way to achieve private visibility:

```rust
use anchor_lang::prelude::*;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        // Call the private_function from within its parent module
        some_function_function::private_function();

        Ok(())
    }

    pub mod some_function_function {
        pub(in crate::func_visibility) fn private_function() {
            // Private function logic...
        }
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

The `pub(in crate::func_visibility)` keyword indicates that `private_function` function is only visible within `func_visibility` module.

We were able to call `private_function` successfully in the initialize function because the initialize function is within `func_visibility` module. Let's try to call `private_function` from outside the module:

```rust
use anchor_lang::prelude::*;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        // Call the private_function from within its parent module
        some_private_function::private_function();

        Ok(())
    }

    pub mod some_private_function {
        pub(in crate::func_visibility) fn private_function() {
            // Private function logic...
        }
    }
}

mod do_something {
    // Import func_visibility module
    use crate::func_visibility;

    pub fn some_func_here() {
        // Call the private_function from outside its parent module
        func_visibility::some_private_function::private_function()

        // Do something...
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

Build the program. What happened? We got an error:

❌ error[E0624]: associated function `private_function` is private

This shows that `private_function` is not publicly accessible and can not be invoked from outside the module where it is visible. Check out [<ins>visibility and privacy</ins>](https://doc.rust-lang.org/beta/reference/visibility-and-privacy.html#pubin-path-pubcrate-pubsuper-and-pubself) in Rust docs for more the pub visibility keyword.

## Contract Inheritance
Direct translation of Solidity contract inheritance to Solana is not possible because Rust does not have classes.

However, a workaround in Rust involves creating separate modules that define specific functionality and then use those modules within our main program, thereby achieving something similar to Solidity’s contract inheritance.

### Getting modules from another file
As programs get larger, we generally don’t want to put everything into one file. Here’s how we can organize logic into multiple files.

Let’s create another file in the `src` folder called `calculate.rs` and copy the provided code into it.

```rust
pub fn add(x: u64, y: u64) -> u64 {
	// Return the sum of x and y 
    x + y
}
```

This add function returns the sum of x and y.

And this, into `lib.rs`.

```rust
use anchor_lang::prelude::*;

// Import `calculate` module or crate
pub mod calculate;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn add_two_numbers(_ctx: Context<Initialize>, x: u64, y: u64) -> Result<()> {
        // Call `add` function in calculate.rs
        let result = calculate::add(x, y);

        msg!("{} + {} = {}", x, y, result);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

In the program above, we imported the calculate module that was created earlier and declared a function called `add_two_numbers` that adds two numbers and logs the result. The `add_two_numbers` function calls the add function in the calculate module, passing `x` and `y` as arguments, then stores the return value in the result variable. The `msg!` macro logs the two numbers that was added and the result.

### Modules don't have to be separate files
The follow example declares a module inside `lib.rs` instead of `calculate.rs`.

```rust
use anchor_lang::prelude::*;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn add_two_numbers(_ctx: Context<Initialize>, x: u64, y: u64) -> Result<()> {
        // Call `add` function in calculate.rs
        let result = calculate::add(x, y);

        msg!("{} + {} = {}", x, y, result);

        Ok(())
    }
}

mod calculate {
    pub fn add(x: u64, y: u64) -> u64 {
		// Return the summation of x and y
        x + y
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

This program does the same as the previous example, with the only difference being that the add function is present in the lib.rs file and within the calculate module. Also, adding the `pub` keyword to a function is crucial, as it makes the function publicly accessible. The code below won’t compile:

```rust
use anchor_lang::prelude::*;

declare_id!("53hgft52DHUKMPHGu1kusuwxFGk2T8qngwSw2SyGRNrX");

#[program]
pub mod func_visibility {
    use super::*;

    pub fn initialize(_ctx: Context<Initialize>) -> Result<()> {
        // Call the private-like function
        let result2 = do_something::some_func_here();

        msg!("The result is {}", result2);

        Ok(())
    }
}

mod do_something {
    // private-like function. It exists in the code, but not everyone can call it
    fn some_func_here() -> u64 {
        // Do something...

        return 20;
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

## Summary
In Solidity, we think a lot about function visibility because it’s very critical. Here’s how to think about it in Rust:
- **Public / External Functions**: These are functions accessible both within and outside the program. In Solana, all functions declared are, by default, public. Everything in the `#[program]` block must be declared `pub`.
- **Internal Functions**: These are functions accessible within the program itself and programs that inherit it. Functions inside a nested `pub` mod block are not included in the built program, but still, they can be accessed within or outside the parent module.
- **Private Functions**: These are functions that are not publicly accessible and cannot be invoked from outside their module. Achieving private visibility in Rust/Solana involves defining a function within a specific module with the `pub(in crate::<module>)` keyword, which makes the function visible within just the module it was defined in.

Solidity achieves contract inheritance through classes, a feature that Rust, the language used in Solana, does not have. Nevertheless, you can still organize your code using Rust modules.

## Learn more with RareSkills
This tutorial is part of our free [Solana course](https://www.rareskills.io/solana-tutorial).