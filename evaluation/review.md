
## A short guide for reviewing Rust packages
This brief document explains the process to identify function calls made to a specific dependency in a Rust package. This is to equip the reader with an explanation of the decision process made to classify the set of 381 cases.

Each line in the [document](evaluation_381_cases.csv) containing the 381 cases has the following format:

| io :: crates :: coco :: v_0_3_4 | io :: crates :: scopeguard :: v_0_3_3 |
|---------------------------------|---------------------------------------|

The first column represents the package and the second column represents one of its specific dependencies. To study this dependency relationship, we need to have an uncompressed version of the source code from the package `coco` at version 0.3.4. This can be obtained from running the `prepare()` operation in RustPräzi (which will download all available packages from [crates.io](https://crates.io)) or manually downloaded from the [API](https://crates.io/api/v1/crates/). 

Manually analyzing the source code of Rust packages can be challenging and at times hard to understand. Hence, we created this document to provide a structured way to analyze Rust packages in three steps: 1) finding an import statement and second, 2) finding uses of it and 3) last function calls it. While it may sound simple in principle, there are many catches which we explain in the form of scenarios:

## 1.  Finding the import statement
Without an import statement of a specific dependency, no functionality of the dependency is available in the codebase. The first step is to find the presence of an import statement in the source code. In Rust, an import statement has the following format:


``` rust
extern crate scopeguard;
```

The statement represents a source code declaration of a specific dependency. To find such a statement in the source code, one needs to use a tool such as `ag` or `grep` and execute the following:

``` bash
ag -C scopeguard
```

The result of executing it may look like one of the following scenarios: 


### Scenario 1 - Nothing


```
```

The output is empty, this may strongly indicate that there exists no declaration of it in the source code. However, this could be false. Rust packages can use a different name than the package identifier (i.e., the `crate` name) in import statements. We can find this out by inspecting the `Cargo.toml` of the dependency. In this case, we inspect it for `scopeguard` package at version 0.3.3. We can identify that under the `lib` section of the `Cargo.toml` file, we find the following:

``` toml
[lib]
    name = "sguard"
```    

The library target name is  `sguard` and if we update our search term with `sguard`, the result could yield into something different:

``` rust
extern crate sguard;
```

If the search would still not yield anything, we mark it as _NOT-IMPORTED_ and stop our search here.


### Scenario 2 - Imported in main.rs or lib.rs, does it matter?


``` 
src/main.rs:61:extern crate scopeguard;
```

or

``` 
src/lib.rs:61:extern crate scopeguard;
```

While both the search results look similar, both cases have semantically different meanings in Rust. The `main.rs` file defines the default binary application of a package and does in principle not export functionality to other parts of the codebase. Hence, the code in a `main.rs` is not reachable to the library code of a package (unless explicitly defined). If the import statement does not appear in any library code (e.g., `lib.rs`), we mark this package as _NOT-IN-LIB_.


In the case of `lib.rs`, we have an actual import of the library functions and we should continue to investigate it further.


### Scenario 3 - Imported in a non-default file?

``` 
src/coco.rs:61:extern crate scopeguard;
```

Sometimes cases like this occur, the package is declared is a non-default file (i.e., not a `lib.rs` or a `main.rs`). To know whether this is used in a library or binary context, we have to investigate the `Cargo.toml` file, which could display something like this:

``` toml
[lib]
src = "src/coco.rs"
```

or 

``` toml
[[bin]]
name = "coco"
src = "src/coco.rs"
```

The former case tells us that this the main entry point to the library functionality of the package and the latter informs us that this is an entry point to a binary application. Depending on the context, we classify similarly to Scenario 2


### Scenario 4 - What about imports in modules?


```
src/core/engine.rs:4 extern crate scopeguard;
```

Here we have an import statement in something that looks like a module. We have to identify whether the module is imported/exported by either a `lib.rs` or a `main.rs`. We can find this out by searching for:

```
ag "mod core" lib.rs / ag "mod core" main.rs
```

Depending on the result, we can establish whether the module is reachable from `lib.rs` or `main.rs`. We classify it similar to Scenario 2 


### Scenario 5 - Headers or attributes on top of import statements

```
60: #[macro_use(defer)]
61: extern crate scopeguard;
```

A header on top of an import statement is a directive to load macro features of a package into the source code. Thus, we need search for invocation or use of macro features in the package. By consulting [docs.rs](https://docs.rs), we can from the documentation of the package retrieve the name of the macros and use those as search terms to find calls or uses of it. In the example, we would look up the documentation for `scopeguard` version `0.3.3` and look at the macro section of the documentation. We can identify that there is one macro function named `defer!`. We grep for `defer!` in the source code. If we find invocations of it, we mark it as a _MACRO_ and if there is no invocation we mark it as _NOT-IMPORT_ (since it is only declared). There are cases in which macros represents an auto-implementation of Traits for certain structs. In such cases, one needs to inspect the macro implementation of the dependency and investigate if the functionality of the dependency is used. For example, there could be cases where just standard library functionality is wrapped in a macro without using anything from the codebase of the package.

Sometimes the documentation can be incomplete and no references to macro. As an additional step, we visit the linked GitHub repository or dig into the code base of the dependency.  

### Scenario 6 - `#[cfg]` next to the import statement

```
#[cfg(feature="guard")] extern crate scopeguard;
```

Due to conditional compilation, the dependency is not part of the default behavior of the package. We mark this as _CONDITIONAL_.

### Scenario 7 - left-pad of import statements

``` rust
        extern crate scopeguard;
```

If there is a large offset of the import statement, this is an indication that this may be
in a code comment. If this is the case, we mark it as _NOT-IMPORT_


## 2. Finding uses of the dependency

Once we have established that there is a clear import statement of the specific dependency in the library code of a package, we need to find cases where the dependency is being used. In general, this can manifest as:


``` rust
use scopeguard::{MudGuard, Utils};
```

or 

``` rust
fn shield(guard: scopeguard::MudGuard, value: i32) -> Result<scopeguard::MudGuard> { ... }
```

The latter example, this is a direct use of the package in other expressions or statements in the code. In the former example, we need to learn how `MudGuard` (either a `Struct` or `Trait`) is being used and how it is being used. By exploring the code base, we might encounter the following scenarios:


### Scenario 1 - just imported

``` rust
use scopeguard::{MudGuard, Utils};
```

We only fund such a statement but `MudGuard` or `Utils` is not used anywhere. Thus, we mark this as _NOT-IMPORT_ (e.g., not used anywhere, just declared)

### Scenario 2 - We find that `MudGuard` is a `Trait` (from the source code or looking at docs.rs)

If it is a `Trait`, we look for implementations of it and further whether the implementations are instantiated and called.  For example, 

``` rust

struct Cycle;

impl MudGuard for Cycle {

    fn serial_number() -> UUID {
        ...
    }
}

fn verify_mudguard(cycle: Cycle) -> Bool {
    cycle::serial_number();
}

```

We can identify that `Cycle` implements `MudGuard` and that there is a concrete instantiation and call of it in the function `verify_mudguard`. If we encounter this particular situation, we continue we establishing the call convention of the invocation. However, if there is no instantiation or no call (e.g., if `verify_mudguard` did not exists in the example), this would be a data dependency and we mark it as a _TYPE-ONLY-DEP_.


### Scenario 3 - We find that `MudGuard` is a `Struct` (from the source code or looking at docs.rs)

if it is a `Struct`, we look for concrete calls of it. This can look like:


``` rust
MudGuard::shield();
```

or 

``` rust
fn shield_me(m : MudGuard) -> Result<...> {
...
  m.shield();
...
}
```


## 3. Determining the call convention 

From the previous step, we are able to find concrete uses of exported `Struct`, `Traits` and function calls belonging to a dependency. We need to now establish the calling convention of it. The examples below will explain this: 


### Scenario 4 - How does a dynamic call look like?

``` rust
fn shield_me(m : &MudGuard) -> Result<...> {
...
  m.shield();

  let x: &Guard = &MudGuard;
  x.shield()
...
}
```
Whenever there is a reference, we have a strong indication that this is a dynamically dispatched function. Currently, the LLVM generator is not able to handle such cases, hence, we mark it as _DYN-CALL_

### Scenario 5 - How does a function call in a generic look like?


``` rust
fn shield_me<T>(m : MudGuard, g: T) -> Result<...> {
...
  m.shield();
...
}
```

Due to the generic function definition, the LLVM call graph generator is not able to analyze the chain of calls inside the function. We also know that the generic definition is not used (i.e., there is no instantiation of it within crates.io ecosystem). Hence, we mark this as a _CALL-IN-GEN-DEF_

### Scenario 6 - unsafe or c-calls

``` rust

extern crate memchr;
extern crate libc;
extern crate pkgA;

use memchr:memchr;


memchr()
libc::atoi()

unsafe {
    ...
    ...
    pkgA::calling();
}

```
All these cases make direct use of hardware or wrap around foreign libraries (e.g., C libraries). In the case of `memchr` or `libc`, the calls are present in the call graph. However, since we need don't support them in our UFI process, they are omitted. In the case of the unsafe block, they involve accessing hardware or wrapping unsafe libraries. These are not shown in the library. We mark this as _C-CALL_.

### Scenario 7 - Headers on top of function definitions?

``` rust

#[cfg(feature = "linear-map")] 
fn shield_me<T>(m : MudGuard, g: T) -> Result<...> {
...
  m.shield();
...
}
```
A `cfg` flag indicates that a certain feature needs to pass during compilation to be able to use the function. Since we are only able to compile with the default behavior, we miss such cases and hence, we mark this as a _CONDITIONAL_


### Scenario 8 - everything looks normal

Typically this occurs when the call graph of the dependency package are missing generic functions due to conditional compilation and unsafe implementation of Traits. We mark such packages as _GENERIC-FUNC-DF_.


## 4. Difficulties in understanding the code
There are cases in which more context is needed or when you are in doubt, the best thing is to read on [docs.rs](https://docs.rs) or visit the GitHub repository of the project. Moreover, to understand language-specific aspects, we refer to the official language reference, [Rust language reference](https://doc.rust-lang.org/stable/reference/)
