# Rust Structs and Attribute-like and Custom Derive Macros

![Rust attribute and custom-derive macros](https://static.wixstatic.com/media/935a00_a473f79694fc407290ce620204641d3c~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a473f79694fc407290ce620204641d3c~mv2.jpg)

Attribute-like and custom derive macros in Rust are used to take a block of Rust code and modify it in some way at compile time, often to add functionality.

To understand attribute-like and custom derive macros in Rust, we first need to briefly cover implementation structs in Rust.

## Implementations for structs: `impl`

The following struct should be straightforward to understand. What gets interesting is when we create functions that operate on a particular struct. The way we do this is with `impl`:

```rust
struct Person {
    name: String,
    age: u8,
}
```

Associated functions and methods are implemented for structs inside the `impl` block.

Associated functions can be compared to the scenario in Solidity where a library is created for interacting with a struct. When we define `using lib for MyStruct`, it allows us to use the syntax `myStruct.associatedFunction()`. This gives the function access to `myStruct` via the `Self` keyword.

We recommend using the [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021) but for more complex examples, you may have to set up your IDE.

Let's look at an example below:

```rust
struct Person {
    age: u8,
    name: String,
}

// Implement a method `new()` for the `Person` struct, allowing initialization of a `Person` instance
impl Person {
    // Create a new `Person` with the provided `name` and `age`
    fn new(name: String, age: u8) -> Self {
        Person { name, age }
    }
    
    fn can_drink(&self) -> bool {
        if self.age >= 21 as u8 {
            return true;
        }
        return false;
    }

    fn age_in_one_year(&self) -> u8 {
        return &self.age + 1;
    }
} 

fn main() {
    // Usage: Create a new `Person` instance with a name and age
    let person = Person::new(String::from("Jesserc"), 19);

    // use some impl functions
    println!("{:?}", person.can_drink()); // false
    println!("{:?}", person.age_in_one_year()); // 20
    println!("{:?}", person.name);
}
```

Usage:

```rust
// Usage: Create a new `Person` instance with a name and age
let person = Person::new(String::from("Jesserc"), 19);

// use some impl functions
person.can_drink(); // false
person.age_in_one_year(); // 20
```

## Rust Traits

Rust traits are a way to implement shared behavior among different `impl`s. Think of them like an interface or abstract contract in Solidity — any contract that uses the interface must implement certain functions.

For instance, let's say we have a scenario where we need to define a Car and Boat struct. We want to attach a method that allows us to retrieve their speed in kilometers per hour. In Rust, we can accomplish this by using a single trait and sharing the method between the two structs.

This is shown below:

```rust
// Traits are defined with the `trait` keyword followed by their name
trait Speed {
    fn get_speed_kph(&self) -> f64;
}

// Car struct
struct Car {
    speed_mph: f64,
}

// Boat struct
struct Boat {
    speed_knots: f64,
}

// Traits are implemented for a type using the `impl` keyword as shown below
impl Speed for Car {
    fn get_speed_kph(&self) -> f64 {
        // Convert miles per hour to kilometers per hour
        self.speed_mph * 1.60934
    }
}

// We also implement the `Speed` trait for `Boat`
impl Speed for Boat {
    fn get_speed_kph(&self) -> f64 {
        // Convert knots to kilometers per hour
        self.speed_knots * 1.852
    }
}

fn main() {
    // Initialize a `Car` and `Boat` type
    let car = Car { speed_mph: 60.0 };
    let boat = Boat { speed_knots: 30.0 };

    // Get and print the speeds in kilometers per hour
    let car_speed_kph = car.get_speed_kph();
    let boat_speed_kph = boat.get_speed_kph();

    println!("Car Speed: {} km/h", car_speed_kph); // 96.5604 km/h
    println!("Boat Speed: {} km/h", boat_speed_kph); // 55.56 km/h
}
```

## How macros can modify structs

In our tutorial on function-like macros, we saw how macros can expand code like `println!(...)` and `msg!(...)` in large Rust code. The other kind of macros we care about in the context of Solana is the *attribute-like* macro and the *derive* macro. We can see all three (function-like, attribute-like, and derive) macros in the starter program anchor creates:

![Rust attribute and custom-derive macros](https://static.wixstatic.com/media/935a00_7626044154134115afa260e9896b370d~mv2.png/v1/fill/w_1480,h_620,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_7626044154134115afa260e9896b370d~mv2.png)

To get an intuition for what the attribute-like macros is doing, we will create two macros: one to add fields to a struct and another that removes them.

### Example 1: attribute-like macro, inserting fields

To gain a better understanding of how Rust attributes and macros work, we will create an [attribute-like macro](https://doc.rust-lang.org/book/ch19-06-macros.html) that:

1. takes a struct which does not have the fields `foo` and `bar`, of type `i32`
2. inserts those fields into the struct
3. creates an `impl` with a function called `double_foo` which returns returns twice the integer value of whatever `foo` is holding.

#### Setup

First we create a new Rust project:

```bash
cargo new macro-demo --lib 
cd macro-demo
touch src/main.rs
```

Add the following to the Cargo.toml file:

```toml
[lib]
proc-macro = true

[dependencies]
syn = {version="1.0.57",features=["full","fold"]}
quote = "1.0.8"
```

#### Creating the main program

Paste the following code into src/main.rs. Be sure to read the comments:

```rust
// src/main.rs
// Import the macro_demo crate and bring all items into scope with the `*` wildcard
// (basically everything in this crate, including our macro in `src/lib.rs`
use macro_demo::*;

// Apply the `foo_bar_attribute` procedural attribute-like macro we created in `src/lib.rs` to `struct MyStruct`
// The procedural macro will generate a new struct definition with specified fields and methods
#[foo_bar_attribute]
struct MyStruct {
    baz: i32,
}

fn main() {
    // Create a new instance of `MyStruct` using the `default()` method
    // This method is provided by the `Default` trait implementation generated by the macro
    let demo = MyStruct::default();

    // Print the contents of `demo` to the console
    // The `Debug` trait implementation generated by the macro allows formatted output with `println!`
    println!("struct is {:?}", demo);

    // Call the `double_foo()` method on `demo`
    // This method is generated by the macro and returns double the value of the `foo` field
    let double_foo = demo.double_foo();

    // Print the result of calling `double_foo` to the console
    println!("double foo: {}", double_foo);
}
```

Some observations:

- The struct `MyStruct` does not have the fields `foo` in it.
- The function `double_foo` is not defined anywhere in the code above, it is assumed to exist.

Now let's create the attribute-like macro which will modify the `MyStruct` behind the scenes.

Replace the code in src/lib.rs with the following code (be sure to read the comments):

```rust
// src/lib.rs
// Importing necessary external crates
extern crate proc_macro;
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemStruct};

// Declaring a procedural attribute-like macro using the `proc_macro_attribute` directive
// This makes the macro usable as an attribute

#[proc_macro_attribute]
// The function `foo_bar_attribute` takes two arguments:
// _metadata: The arguments provided to the macro (if any)
// _input: The TokenStream the macro is applied to
pub fn foo_bar_attribute(_metadata: TokenStream, _input: TokenStream) -> TokenStream {
    // Parse the input TokenStream into an AST node representing a struct
    let input = parse_macro_input!(_input as ItemStruct);
    let struct_name = &input.ident; // Get the name of the struct

    // Constructing the output TokenStream using the quote! macro
    // The quote! macro allows for writing Rust code as if it were a string,
    // but with the ability to interpolate values
    TokenStream::from(quote! {
        // Derive Debug trait for #struct_name to enable formatted output with `println()`
        #[derive(Debug)]
        // Defining a new struct #struct_name with two fields: foo and bar
        struct #struct_name {
            foo: i32,
            bar: i32,
        }

        // Implementing the Default trait for #struct_name
        // This provides a default() method to create a new instance of #struct_name
        impl Default for #struct_name {
            // The default method returns a new instance of #struct_name
            // with foo set to 10 and bar set to 20
            fn default() -> Self {
                #struct_name { foo: 10, bar: 20}
            }
        }

        impl #struct_name {
            // Defining a method double_foo for #struct_name
            // This method returns double the value of foo
            fn double_foo(&self) -> i32 {
                self.foo * 2
            }
        }
    })
}
```

Now, to test our macro we run the code in main.rs with `cargo run src/main.rs`.

We get this output:

```bash
struct is MyStruct { foo: 10, bar: 20 }
double foo: 20
```

### Example 2: an attribute-like macro, removing fields
The best way to think about attribute-like macros is that they have unlimited power in how they modify the struct. Let's repeat the example above, but this time the attribute-like macro will remove all the fields from the struct.

Replace src/lib.rs with the following:

```rust
// src/lib.rs
// Importing necessary external crates
extern crate proc_macro;
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemStruct};

#[proc_macro_attribute]
pub fn destroy_attribute(_metadata: TokenStream, _input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(_input as ItemStruct);
    let struct_name = &input.ident; // Get the name of the struct

    TokenStream::from(quote! {
        // This returns an empty struct with the same name
        #[derive(Debug)]
        struct #struct_name {
        }
    })
}
```

Replace src/main.rs with the following:

```rust
use macro_demo::*;

#[destroy_attribute]
struct MyStruct {
    baz: i32,
    qux: i32,
}

fn main() {
    let demo = MyStruct { baz: 3, qux: 4 };

    println!("struct is {:?}", demo);
}
```

When you try to compile it with `cargo run src/main.rs` you will get the following error:

![Error: struct `MyStruct` has no field named `baz`](https://static.wixstatic.com/media/935a00_12dd38aa02e64ed380bc5e5638f26c35~mv2.png/v1/fill/w_1480,h_764,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_12dd38aa02e64ed380bc5e5638f26c35~mv2.png)

It may seem odd, because the struct clearly has those fields. However, the attribute-like macro removed them!

## The `#[derive(…)]` macro

The `#[derive(…)]` macro is much less powerful than the attribute-like macro. For our purposes, a derive macro *augments* a struct, it does not alter it. (This is not a precise definition, but it is sufficient for now).

A derive macro can, among other things, attach an `impl` to a struct.

For example, if we try to do the following:

```rust
struct Foo {
    bar: i32,
}

pub fn main() {
    let foo = Foo { bar: 3 };
    println!("{:?}", foo);
}
```

The code will not compile because structs are not "printable."

To make them printable, they need an `impl` with a function `fmt` which returns a string representation of the struct.

If we do the following instead:

```rust
#[derive(Debug)]
struct Foo {
    bar: i32,
}

pub fn main() {
    let foo = Foo { bar: 3 };
    println!("{:?}", foo);
}
```

We expect it to print:

```bash
Foo { bar: 3 }
```

The derive attribute "augmented" `Foo` in such a way that `println!` could create a string representation for it.

## Summary
An `impl` is a group of functions that operate on a struct. They are "attached" to the struct by using the same name as the struct. A trait enforces that an `impl` implements certain functions. In our example, we attached the the trait Speed to `impl` Car using the syntax `impl` Speed for Car.

An attribute-like macro takes in a struct and can completely rewrite it.

A derive macro augments a struct with additional functions.

### Macros allow Anchor to hide complexity
Let's look at the program anchor creates during anchor init again:

![Rust attribute and custom-derive macros](https://static.wixstatic.com/media/935a00_7626044154134115afa260e9896b370d~mv2.png/v1/fill/w_1480,h_620,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_7626044154134115afa260e9896b370d~mv2.png)

The attribute `#[program]` is modifying the module behind the scenes. For example, it implements a router that automatically directs incoming blockchain instructions to the appropriate functions within the module.

The struct `Initialize {}` is augmented with additional functionality to be used in the Solana framework.

## Summary
Macros a very large topic. Our intent here is to give you a sense of what is happening when you see `#[program]` or `#[derive(Accounts)]`. Don't be discouraged if it feels foreign. **You do not need to be able to write macros to write Solana programs**.

Having an idea of what they do however will hopefully make the programs you see less mysterious.

## Learn more with RareSkills
This tutorial is part of our free [Solana course](https://www.rareskills.io/solana-tutorial).

*Originally Published February, 16, 2024*