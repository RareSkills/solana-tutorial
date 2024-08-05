# The unusual syntax of Rust

![The unusual syntax of Rust](https://static.wixstatic.com/media/706568_0ea13ed362d34a8cab96a2198c15d40f~mv2.jpg/v1/fill/w_1480,h_832,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_0ea13ed362d34a8cab96a2198c15d40f~mv2.jpg)

Readers coming from a Solidity or Javascript background may find Rust’s usage and syntax of `&`, `mut`, `<_>`, `unwrap()`, and `?` to be weird (or even ugly). This chapter explains what these terms mean.

Don't worry if everything doesn't sink in right away. You can always come back to this tutorial later if you forget the syntax definitions.

## Ownership `&` Borrowing (references & deref operator `*`):
### Rust Copy Type
To understand `&` and `*` we first need to understand the “copy type” in Rust. A copy type is a datatype that is small enough that the overhead of copying the value is trivial. The following values are copy types:
- integers, unsigned, and floats integers
- booleans
- char

The reason they are “copy types” is because they have a small fixed size.

On the other hand, vectors, strings, and structs can be arbitrarily large, so they are not copy types.

### Why Rust makes a distinction between copy types and non-copy types
Consider the following Rust code:

```rust
pub fn main() {
	let a: u32 = 2;
	let b: u32 = 3;
	println!("{}", add(a, b)); // a and b a are copied to the add function
	
	let s1 = String::from("hello");
	let s2 = String::from(" world");

	// if s1 and s2 are copied, this could be a huge data transfer
  // if the strings are very long
	println!("{}", concat(s1, s2));
}

// implementations of add() and concat() are not shown for brevity
// this code does not compile
```

In the first section of code where `a` and `b` are added, only 64 bits of data need to be copied from the variables to the function (32 bits * 2 variables).

*However in the case of the string, we don’t always know in advance how much data we are copying*. If the string was 1 GB long, the program would lag significantly.

Rust wants us to be explicit about how we want large data to be handled. It will not copy it behind the scenes like how dynamic languages do.

Therefore, when we do something as simple as *assigning a string to a new variable* Rust will do something many find to be unexpected as we will see in the next section.

## Ownership in Rust
For non-copy types (Strings, vectors, structs, etc), once the value is assigned to the variable, that variable “owns” it. The implications of ownership will be demonstrated shortly.

The following code will not compile. The explanation is in the comments:

```rust
// Example of changing ownership on a non-copy datatype (string)
let s1 = String::from("abc");

// s2 becomes the owner of `String::from("abc")`
let s2 = s1;

// The following line will fail to compile because s1 can no longer access its string value.
println!("{}", s1);

// This line compiles successfully because s2 now owns the string value.
println!("{}", s2);
```

To fix the code above we have two options: use the `&` operator or clone `s1`.

### Option 1: give `s2` a view of `s1`

In the code below, note the important & prepending s1:

```rust
pub fn main() {
    let s1 = String::from("abc");
	
	let s2 = &s1; // s2 can now view `String::from("abc")` but not own it
	
	println!("{}", s1); // This compiles, s1 still holds its original string value.
	println!("{}", s2); // This compiles, s2 holds a reference to the string value in s1.
}
```

If we want another variable to “view” the value (i.e. get read-only access), we use the `&` operator.

**To give another variable or function a view of an owned variable, we prepend it with `&`.**

It may be helpful to think of `&` as “view only” mode for a non-copy type. The technical word for what we are calling “view only” is **borrowing**.

### Option 2: clone `s1`
To understand how we might clone a value, consider the following example:

```rust
fn main() {
    let mut message = String::from("hello");
    println!("{}", message);
    message = message + " world";
    println!("{}", message);
}
```

The code above will print “hello” and then “hello world” as expected.

If we add another variable `y` that views message however, the code will no longer compile:

```rust
// Does not compile
fn main() {
    let mut message = String::from("hello");
    println!("{}", message);
    let mut y = &message; // y is viewing message
    message = message + " world";
    println!("{}", message);
    println!("{}", y); // should y be "hello" or "hello world"?
}
```

Rust does not accept the code above, because the variable message cannot be reassigned while it is being viewed.

If we want y to be able to copy the value of message without interfering with message down the line, we can instead clone it:

```rust
fn main() {
    let mut message = String::from("hello");
    println!("{:?}", message);
    let mut y = message.clone(); // change this to clone
    message = message + " world";
    println!("{:?}", message);
    println!("{:?}", y);
}
```

The above code will print:

```bash
hello
hello world
hello
```

## Ownership is only an issue with non-copy types
If we replace our String (which is a non-copy type) with a copy type (like an integer), we won't run into any of the issues above. Rust will happily copy the copy type because the overhead is negligible.

```rust
let s1 = 3;

let s2 = s1;

println!("{}", s1);
println!("{}", s2);
```

## The `mut` keyword

By default, all variables are immutable in Rust unless the `mut` keyword is specified.

The following code will not compile:

```rust
pub fn main() {
	let counter = 0;
	counter = counter + 1;

	println!("{}", counter);
}
```

If we try to compile the code above we will get the following error:

![Error: cannot assign twice](https://static.wixstatic.com/media/935a00_9f8e31ed1b7c4262bc1aad799e83067a~mv2.png/v1/fill/w_1254,h_381,al_c,q_90,enc_auto/935a00_9f8e31ed1b7c4262bc1aad799e83067a~mv2.png)

Fortunately, if you forget to include the `mut` keyword, the compiler is usually smart enough to point out the mistake clearly. The following code inserts the mut keyword enabling the code to compile:

```rust
pub fn main() {
	let mut counter = 0;
	counter = counter + 1;

	println!("{}", counter);
}
```

## Generics in Rust: the `< >` syntax
Let’s consider a function that takes a value with an *arbitrary* type and returns a struct with a field `foo` containing that value. Rather than writing a bunch of functions for every possible type, we can use a *generic*.

The example struct below can be an `i32` or a `bool`.

```rust
// derive the debug trait so we can print the struct to the console
#[derive(Debug)]
struct MyValues<T> {
    foo: T,
}

pub fn main() {
    let first_struct: MyValues<i32> = MyValues { foo: 1 }; // foo has type i32
    let second_struct: MyValues<bool> = MyValues { foo: false }; // foo has type bool
    
    println!("{:?}", first_struct);
    println!("{:?}", second_struct);
}
```

Here is why this is handy: when we store values “in storage” in Solana, we want to be very flexible if we are going to store a number, string, or something else.

If our struct had more than one field in it, the syntax to parameterize the types is as follows:

```rust
struct MyValues<T, U> {
    foo: T,
	bar: U,
}
```

Generics is a very large topic in Rust, so by no means are we giving a complete treatment here. However, this is sufficient to get a decent understanding of most Solana programs.

## Options, Enums, and Deref `*`
To show the importance of options and enums, let's consider the following example:

```rust
fn main() {
	let v = Vec::from([1,2,3,4,5]);

	assert!(v.iter().max() == 5);
}
```

The code fails to compile with the following error:

```bash
6 |     assert!(v.iter().max() == 5);
  |                               ^ expected `Option<&{integer}>`, found integer
```

The output of `max()` is not an integer due to the corner case that the vector v might be empty.

### The Rust Option
To handle this corner case, Rust returns an Option instead. An Option is an enum which can contain either the expected value, or a special value that indicates “nothing was there.”

To turn an Option into the underlying type, we use` unwrap()`. `unwrap()` will cause a panic if we received “nothing”, so we should only use it in situations where we want the panic to occur or we are sure we won’t get a empty value.

To make the code work as expected, we can do the following:

```rust
fn main() {
	let v = Vec::from([1,2,3,4,5]);

	assert!(v.iter().max().unwrap() == 5);
}
```

### The deref `*` operator
```bash
19 |     assert!(v.iter().max().unwrap() == 5);
   |                                     ^^ no implementation for `&{integer} == {integer}`
```

The term on the left hand side of the equality is a *view* (i.e. &) of an integer and the term on the right is an actual integer.

To convert a “view” of an integer to a regular integer, we need to use the "dereference" operation. This is when we prepend the value with a `*` operator.

```rust
fn main() {
	let v = Vec::from([1,2,3,4,5]);

	assert!(*v.iter().max().unwrap() == 5);
}
```

Because the elements of the array are copy types, the deref operator will silently copy the 5 returned by `max().unwrap()`.

You can think of `*` as "undoing" a & without disturbing the original value.

*Using the  operator on non-copy types is a complicated subject. For now, all you need to know is that if you receive a view (borrow) of a copy type and need to turn it into the “normal” type, use the `*` operator.*

## `Result` vs Option in Rust
An option is used when we might receive something “empty.” A `Result` (the same `Result` Anchor programs have been returning) is used when we might receive an error.

### Result Enum
The `Result<T, E>` enum in Rust is used when a function's operation may either succeed and return a value of type `T` (a generic type) or fail and return an error of type `E` (generic error type). It is designed to handle operations that can result in either a successful outcome or an error condition.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

In Rust, the `?` operator is used for the `Result<T, E>` enum, while the `unwrap()` is used for both the `Result<T, E>` and the `Option<T> enums`.

## The `?` operator
The `?` operator can only be used in functions that return a `Result` as it is syntactic sugar for returning either and `Err` or `Ok`.

The `?` operator is used to extract data from the `Result<T, E>` enum and return the `OK(T)` variant if the function execution is successful or bubble up an error `Err(E)` if there is an error. The `unwrap()` method works the same way but for both `Result<T, E>` and `Option<T>` enums, however, it should be used cautiously due to its potential for crashing the program if an error occurs.

Now, consider the following code below:

```rust
pub fn encode_and_decode(_ctx: Context<Initialize>) -> Result<()> {
    // Create a new instance of the `Person` struct
    let init_person: Person = Person {
        name: "Alice".to_string(),
        age: 27,
    };

    // Encode the `init_person` struct into a byte vector
    let encoded_data: Vec<u8> = init_person.try_to_vec().unwrap();

    // Decode the encoded data back into a `Person` struct
    let data: Person = decode(_ctx, encoded_data)?;

    // Logs the decoded person's name and age
    msg!("My name is {:?}, I am {:?} years old.", data.name, data.age);

    Ok(())
}

pub fn decode(_accounts: Context<Initialize>, encoded_data: Vec<u8>) -> Result<Person> {
    // Decode the encoded data back into a `Person` struct
    let decoded_data: Person = Person::try_from_slice(&encoded_data).unwrap();

    Ok(decoded_data)
}
```

The `try_to_vec()` method encodes a struct to a byte vector and returns a `Result<T, E>` enum where `T` is the byte vector, while the `unwrap()` method is used to extract the value of the byte vector from `OK(T)`. This will crash the program if the method fails to convert the struct to a byte vector.

## Learn more with RareSkills
This tutorial is part of our free [Solana course](https://www.rareskills.io/solana-tutorial).