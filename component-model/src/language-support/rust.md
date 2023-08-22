# Components in Rust

Rust has first-class support for the component model via [the `cargo component` tool](https://github.com/bytecodealliance/cargo-component).

## Installing `cargo component`

To install `cargo component`, run:

```
cargo install --git https://github.com/bytecodealliance/cargo-component --locked cargo-component
```

> There is currently no binary or `crates.io` distribution of `cargo component`.

## Creating a library component with `cargo component`

See [the language guide](../language-support.md#building-a-component-with-cargo-component).

## Exporting an interface with `cargo component`

The sample `add.wit` file exports a function. However, to use your component from another component, it must export an interface. This results in slightly fiddlier bindings. For example, to implement the following world:

```
package docs:sample

interface demo {
    test: func(text: string) -> string
}

world sample {
    export demo
}
```

you would write the following Rust code:

```rust
cargo_component_bindings::generate!();

// Separating out the interface puts it in a sub-module
use bindings::exports::docs::sample::demo::Demo;

struct Component;

impl Demo for Component {
    fn test(text: String) -> String {
        todo!()
    }
}
```

## Export versioning

When `cargo component build` exports an interface, it sets the version to the package version, as listed in `Cargo.toml`. For example, given the interface above, if `Cargo.toml` contains:

```toml
[package]
name = "sample"
version = "0.2.0"
```

then the interface is exported as `docs:sample@0.2.0`. This can be important when working with binary tools that care about interface versions.

## Importing an interface with `cargo component`

The world file (`wit/world.wit`) generated for you by `cargo component new --reactor` doesn't specify any imports.

> `cargo component build`, by default, uses the Rust `wasm32-wasi` target, and therefore automatically imports any required WASI interfaces - no action is needed from you to import these. This section is about importing custom WIT interfaces from library components.

If your component consumes other components, you can edit the `world.wit` file to import their interfaces.

For example, suppose you have created and built an adder component:

```
// in the 'adder' project

// wit/world.wit
package docs:adder@0.1.0

interface add {
    add: func(a: u32, b: u32) -> u32
}

world adder {
    export add
}

// src/lib.rs
cargo_component_bindings::generate!();

use bindings::exports::docs::adder::add::Add;

struct Component;

impl Add for Component {
    fn add(a: u32, b: u32) -> u32 {
        a + b
    }
}
```

and that you now want to use that component in a calculator component. Here is a partial example world for a calculator:

```
// in the 'calculator' project

// wit/world.wit
package docs:calculator

interface calculate {
    eval-expression: func(expr: string) -> u32
}

world calculator {
    export calculate
    import docs:adder/add@0.1.0
}
```

### Referencing the package to import

Because the `docs:adder` package is in a different project, we must first tell `cargo component` how to find it. To do this, add the following to the `Cargo.toml` file:

```toml
[package.metadata.component.target.dependencies]
"docs:adder" = { path = "../adder/wit" }  # directory containing the WIT package
```

Note that the path is to the adder project's WIT _directory_, not to the `world.wit` file. A WIT package may be spread across multiple files in the same directory; `cargo component` will look at all the files.

### Calling the import from Rust

Now the declaration of `add` in the adder's WIT file is visible to the `calculator` project. To invoke the imported `add` interface from the `calculate` implementation:

```rust
// src/lib.rs
cargo_component_bindings::generate!();

use bindings::exports::docs::calculator::calculate::Calculate;

// Bring the imported add function into scope
use bindings::docs::adder::add::add;

struct Component;

impl Calculate for Component {
    fn eval_expression(expr: String) -> u32 {
        // Cleverly parse `expr` into values and operations, and evaluate
        // them meticulously.
        add(123, 456)
    }
}
```

### Fulfilling the import

When you build this using `cargo component build`, the `add` interface remains imported. The calculator has taken a dependency on the `add` _interface_, but has not linked the `adder` implementation of that interface - this is not like referencing the `adder` crate. (Indeed, `calculator` could import the `add` interface even if there was no Rust project implementing the WIT file.) You can see this by running [`wasm-tools component wit`](https://github.com/bytecodealliance/wasm-tools/tree/main/crates/wit-component) to view the calculator's world:

```
# Do a release build to prune unused imports (e.g. WASI)
$ cargo component build --release

$ wasm-tools component wit ./target/wasm32-wasi/release/calculator.wasm
package root:component

world root {
  import docs:adder/add@0.1.0

  export docs:calculator/calculate@0.1.0
}
```

As the import is unfulfilled, the `calculator.wasm` component could not run by itself in its current form. To fulfil the `add` import, so that the calculator can run, you would need to [compose the `calculator.wasm` and `adder.wasm` files into a single, self-contained component](../creating-and-consuming/composing.md).