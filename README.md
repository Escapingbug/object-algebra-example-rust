# Object Algebra Example in Rust

## What is Object Algebra?

Object Algebra is more like a design pattern, to solve a well-known program design problem called "expression problem".

So what exact problem are we facing? What is expression problem?

### Expression Problem

Let's say we are writing a programming language or something alike.
We will be having a tree-structure (recursive-structure).
To be more simple, let's just assume we are dealing with a programming language.
In this case, [AST (abstract syntax tree)](https://en.wikipedia.org/wiki/AST) is the structure of desire, recursive and "tree-like" (well, actually a real tree).

Our programming language may have a REALLY simple structure that looks like this:

```rust
pub enum AstNode {
    Add(Box<AstNode>, Box<AstNode>),
    Value(i64),
}
```

Simple enough, this "language" works just like a mathmetic expression with only add operation.

Note that how we define our AST tree, since Rust has support for algebraic datatype (it's OK you don't understand this term, you can read "algebraic datatype" as "possibly recursive enum"), we used such a way to define the node which can be recursive.
The "box" is required to define such a recursive structure or Rust cannot determine the actual size of such a data structure.

So, how can we execute this AST? A naive approach would be like this:

```rust
use AstNode::*;

fn interpret(node: Box<AstNode>) -> i64 {
    match *node {
        Add(a, b) => interpret(a) + interpret(b),
        Value(x) => x
    }
}
```

Pretty simple, just a match.

How about we want to print out the tree instead of interpreting?
Still not quite difficult:

```rust
use AstNode::*;

fn print(node: Box<AstNode>) -> String {
    match *node {
        Add(a, b) => format!("({} + {})", print(a), print(b)),
        Value(x) => format!("{}", x)
    }
}
```

Now, wait, we seems to forget the modulo operation, the "%" case.
OK, let's add it back. What will we do?

1. Add a case to the AST node enum
2. Modify the `interpret` function, so that the execution still works by adding a new case
3. Modify the `print` function, so that the printing still works by adding a new case

Three steps, and unfortunately, all destructive.
That is, our modification is not "extensible".
It is not done by "adding" new code but by modifying.
What if the original code is not possible to "modify"?
An example would be, the `interpret` and `print` are both in another crate, `AstNode` is in another crate as well.

This is an example of the "expression problem".
The problem claims several requirement:

1. we can add a new case to existing datatype
2. we can add a new case to existing operations (such as `interpret` and `print`, we might add more)
3. both adding must not require "recompilation" of the existing code. In Rust world, you might understand this as adding cannot modify the referenced crate.
4. type-safety is not broken (or else, you are just free to do anything you like)

### Datatype ala carte

There's already a solution to this problem, called "datatype ala carte".
If you are into more detail of this solution, there's already a [Rust example of this technique](https://github.com/dcreager/expression-problem-rust).

To ease the way of following here and there, we will also talk about the solution here.

Remember the `AstNode` definition?
We now use another form to express the same semantic, in this way:

```rust
pub struct Add<E>(E, E);
pub struct Value(i64);
```

In this step, the "enum" is split apart into structs.
Of course after this step, you don't need to modify any existing "enum" to add a new type.
For example, let's add a "Sub" case:

```rust
pub struct Sub<E>(E, E);
```

Plain and simple.
Let's define some expression this way:

```rust
let a: Add<Value> = Add(Value(1), Value(1)); // works!
let b = Add(Value(1), Add(Value(2), Value(3))); // oops!
```

This won't work! Why? Because, the `E` part within `Add` needs to be the same.
To deal with this problem, let's add one more thing:

```rust
pub enum Sum<L, R> {
    Left(L),
    Right(R)
}
pub type Ops<E> = Sum<Value, Add<E>>;
pub struct AstNode(pub Box<Ops<AstNode>>);
```

First, we have `Sum` type (enum) definition.
This type is exactly like `Result` type, but since it does not represent anything related to error handling, so we redefine it.
The word "sum" is from the algebraic data type, you don't need to care too much about that.

The next thing we have is the `Ops` definition.
It defines the type of the operations, or constructs (yeah, just like the choices in the enum).
So, we have `Value` and `Add` now.

And the final magic, the `AstNode`.

Let's look at how this works, by defining the expressions again:

```rust
let a = AstNode(
    Box::new(Sum::Right(Add::<AstNode>(
            Box::new(Sum::Left(Value(1))),
            Box::new(Sum::Left(Value(1))))))
); // 1 + 1

let b = AstNode(
    Box::new(Sum::Right(Add::<AstNode>(
            Box::new(Sum::Left(Value(1))),
            Box::new(Sum::Right(Add::<AstNode>(
                Box::new(Sum::Left(Value(2))),
                Box::new(Sum::Left(Value(3)))))))))); // 1 + (2 + 3)
```

I hate to admit that, but this ugly representation really works.
There are things you can do to make this less ugly, like macro.
Macro is perfect in such situation, at least in Rust world, when we need such DSL we use macro.
However, there are other options as well.
One example is to use the `From` trait and use the function as the constructor.

For more on how to use `From` trait to simplify this ugly representation, [the original post](https://github.com/dcreager/expression-problem-rust/blob/master/src/ch04_smart_constructors.rs) should help.
I'm not gonna talk about that since it is not the core.

OK, one more thing.
Does this representation actually solves our problem?

To check this out, we need to make sure two things:

1. it's non-destructive to add the operation
2. it's non-destructive to add the datatype

First things first, let's define the first operation, execute:

```rust
/// Once this trait is defined, the signature of any operation
/// others are just strait forward.
pub trait Execute {
    fn exec(&self) -> i64;
}

impl<E> Execute for Value {
    fn exec(&self) -> i64 {
        self.0
    }
}

impl<E> Execute for Add<E>
    where E: Execute {
    fn exec(&self) -> i64 {
        self.0.exec() + self.1.exec()
    }
}

impl<L, R> Execute for Sum<L, R>
where
    L: Execute,
    R: Execute {
    fn exec(&self) -> i64 {
        match self {
            Sum::Left(l) => l.exec(),
            Sum::Right(r) => r.exec()
        }
    }
}

impl Execute for AstNode {
    fn exec(&self) -> i64 {
        // self.0 => Box<Sum<Value, Add<E>>>
        // execute is defined for Sum, so this works
        self.0.exec()
    }
}
```

A bit boilerplate, but yes, this implements the execute operation.
This works by delegate the actual execute implementation to the internal structure.
Since the definition of the structure `AstNode` is recursive, but in the end it must have a concrete struct such as `Value`, so this execute has to end.
In other words, this `Execute` implementation keeps recursing until the final `Value` where we get the value.
And whenever we encounter `Add`, we do the addition.

Ok, enough of this.
What about adding a new operation, such as the `print`?
Let's go for it!

```rust
// (I know there's ToString, just to keep consistent with the `print` function we have defined above.)
pub trait Print {
    fn print(&self) -> String;
}

impl Print for Value {
    fn print(&self) -> String {
        self.0.print()
    }
}

impl<E> Print for Add<E>
where
    E: Print {
    fn print(&self) -> String {
        format!("({} + {})", self.0.print(), self.1.print())
    }
}

impl<L, R> Print for Sum<L, R>
where
    L: Print,
    R: Print {
    fn print(&self) -> String {
        match self {
            Sum::Left(l) => l.print(),
            Sum::Right(r) => r.print()
        }
    }
}

impl Print for Value {
    fn print(&self) -> String {
        format!("{}", self.0)
    }
}
```

Pretty strait forward, right?
Just define the `Print` trait implementation for each of the construct we have.

Now, let's check out if the "adding new data type" rule is followed.
Let's add a subtraction datatype.

```rust
// just like Add
pub struct Sub<E>(E, E);
```

And let's see how this would interfere with the operation:

```rust
impl<E> Execute for Sub<E>
    where E: Execute {
    fn exec(&self) -> i64 {
        self.0.exec() - self.1.exec()
    }
}

impl<E> Print for Sub<E>
    where E: Print {
    fn print(&self) -> i64 {
        format!("({} - {})", self.0.print(), self.1.print())
    }
}
```

So far so good, define the operation for new datatype is easy.
How to now add the construct?
We now have some choices.
In fact, the `AstNode` defined above is a specialized version of `AstNode` that contains `Add` and `Value`.
So, we need to support `Sub` now.
If we'd like to keep using `AstNode`, we now have to modify it.
Or, we can make each of the "Node" specialized to what it supports.

```rust
// redefine `AstNode` to its specialized version now
pub struct AddAstNode(pub Box<Ops<AddAstNode>>);
// then we are possible to have a sub supported one
pub type AddSubOps<E> = Sum<Sub<E>, Ops<E>>; // add the sub to the game!
pub struct AddSubAstNode(pub Box<AddSubOps>);
```

TODO: the rest -- add the expression example
TODO: add object algebra example