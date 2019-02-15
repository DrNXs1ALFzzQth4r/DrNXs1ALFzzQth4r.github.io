
# RustPräzi

A Rust implementation of Präzi for constructing call-based dependency networks of [crates.io](https://crates.io/). The prototype is developed in Rust with certain functionality written in Python. In future releases, the Python scripts will be re-written to Rust code. 

## Installation Prerequisites

- The Rust toolchain with `rustup` (download at the [official website](https://www.rust-lang.org/en-US/install.html))
- A pre-built binary of LLVM 4.0 (download at [official website](http://releases.llvm.org/download.html#4.0.0)). In the `config.ini` (root of the repository), specify the path to the uncompressed LLVM binary.
- Recommended OS: Ubuntu 16.04.3 LTS

## Runtime environment

RustPräzi is developed to run in a parallel execution environment such as a cluster. 

## Constructing a Package-based dependency network (PDN)

1. Clone the [rust-lang/crates.io-index](https://github.com/rust-lang/crates.io-index) repository
2. To build from a specific revision/time point of the index, use the `git checkout <commit-revsion>` command
3. In `src/bin/graph.rs`, specify the location of the index in the `repo_path()` function as:

``` rust
fn repo_path() -> PathBuf {
    PathBuf::from("/path/to/crates_index")
}
```
4. To create a PDN from the index, run the following

``` bash
cargo run --bin gen-pdn-graph --release > edges.json
```

The output is a set of edges representing the derived dependency network from the index. Due to a problem with the serializer, the set of nodes is produced by uncommenting the section below in `graph.rs` (and comment out the section with edges):

``` rust
    //  let result = json!({
    //     "nodes": &node_table
    //  });
```

## Building packages and call graphs from the index

There are three specific files involved for downloading, building and generating call graphs from the index. 

- `src/core/crates.rs` - hosts the core functionality to perform each step in parallel
- `src/core/callgraph.rs` - Rust wrapper of LLVM's call graph generator and Cargo's build chain (`crates.rs` use this functionality)
- `src/main.rs` - main entry point to invoke a specific process from the `crates.rs` file

### 1. Downloading packages

#### Initialize

Internally, `crates.rs` creates a list of package versions of an index snapshot and stores them in a `CratesList` struct. The location of the index is specified in `src/utils/crates_index.rs`. The `prepare()` function performs the task of downloading and uncompressing packages. First, it populates index entries in a  `CratesList` and then for each entry in the list, download and uncompress the package version. To run this step, one needs to add the following code snippet in `src/main.rs`:

``` rust
  match prepare() {
        Ok(_) => info!("Done!"),
        Err(e) => error!("We failed with the error {}!", e),
    }
```    

and then run as `cargo run --bin`. The output will report the number of successful downloads and error messages.

#### Validating `Cargo.toml` files
As part of the submission process to `crates.io`, Cargo validates a set of fields but leave other fields unvalidated. To identify the number of broken build manifests, the `readable_manifest()` function performs syntax checks of all package versions in the `CratesList`. Internally, it uses Cargo's manifest validator. To run it, 

``` rust
  match readable_manifest() {
         Ok(_) => info!("Done!"),
        Err(e) => error!("We failed with the error {}!", e),
     }
```
and then run as `cargo run --bin`. The output will report the number of broken manifests along with their error message.


#### How many package versions are libraries?
The `islibrary()` function reads the build targets of all build manifests in the `CratesList` and filter those who don't produce a library artifact. To run it, 

``` rust
  match isLibrary() {
        Ok(_) => info!("Done!"),
         Err(e) => error!("We failed with the error {}!", e),
    }
```
and then run as `cargo run --bin`. The output will report number of package versions which do not produce a library


### 2. Building packages and creating call graphs

#### Compiling

To infer a call graph from each package version, we need to build each package version to obtain the LLVM bytecode file (i.e., the LLVM Bitcode file). The `build_crate()` will attempt build package versions from the `CratesList`. The function will also avoid rebuilding package versions if there exist a `.bc` file in the `target` folder. Thus, the function can be re-run without any problems. `build_crate()` will by default use the installed and selected `rustc` version of the system. To use a different compiler version, the user needs to select a different compiler version from `rustup` and then re-run this step.

#### Re-writting build manifests

A problem with many build manifests is that they use a local development source of a dependency hosted on [crates.io](https://crates.io). To fix this, we developed a function called `normalize_manifests()` which will re-write dependency sources to use [crates.io](https://crates.io) for downloading the dependency.

#### Creating call graphs

The `build_callgraph()` will construct a call graph from each package version which has an LLVM call graph `.bc` file. A `callgraph.dot` file is generated and stored in the root of the package folder.


### 3. Build log analyzer

As an optional step, the `src/utils/build_log_analyzer.py` parses generated compilation errors and output statistics of the failures into several categories. This can be useful to get an overview of common compilation errors.


## De-mangling, pruning, UFIying call graphs and merging them into the final RustPrāzi

At this phase, we have downloaded, built and constructed call graphs for the package versions specified in `CratesList`. The remaining steps involves pruning the call graphs, constructing unique identifiers for function nodes in each call graph and then finally merging all call graphs into a single RustPräzi instance. These are the steps involved for this process (together with the input and output of each step):

1.  __Demangling__ - {Input(`callgraph.dot`, Output(`callgraph.unmangled.graph`)}
2.  __Pruning__ - {Input(`callgraph.unmangled.graph`), Output(`callgraph.unmangled.pruned.graph`)}
3.  __UFIfy__ - {Input(`Cargo.lock`, `callgraph.unmangled.pruned.graph`), Output(`callgraph.ufi.graph`)}
4.  __Merging callgraphs to a RustPräzi__ - {Input(`callgraph.ufi.graph`), Output(`callgraph.ufi.merged.graph`)}
5.  __Constructing a RustPräzi PDN__ - {Input(`callgraph.ufi.merged.graphh`), Output(`crate.dependency.callgraph.graph`)}

The `utils/ufi.sh` performs all the steps above in one single shell script. Before running the script, the following needs to be executed:

``` bash
    pip install -r src/requirements.txt 
    cargo build --bin ufi --release
```

Below is a brief explanation of each step:

### 1. Demangling
Demangling is handled by `rustfilt`, it takes an identifier such as `_ZN37_$LT$mio..notify..Notify$LT$M$GT$$GT$13with_capacity17h61bf4481458ad279E` and converts into a human-readable format  such as `<mio::notify::Notify<M>>::with_capacity`. 

### 2 Pruning
The `src/prune_cg.py` script removes redundant nodes and edges in the call graph. We remove both the `null` and `external` node along with its edges.


### 3. Ufiying
The `src/bin/ufi.rs` contains the functionality to construct a UFI from a Rust-demangled function identifier. A pseudo-pythonic-way of the algorithm is shown below:

```
    ast = syn::parse(identifier)  ///using out syn library
    for namespace in ast:
        pkg_id = extract(namespace)
        ver = lookup_dep(pkg_id)
        namespace.prepend(ver)
        namespace.prepend(crates)
        namespace.prepend(io)
    ufi = syn::tokenize(ast)
    replace_in_callgraph(identifier, ufi)
```

The namespaces are also classified whether they belong to the package itself, dependency or a Rust library. The following labels are attached to each namespace:

``` rust
enum Symbol {
    InternalCrate,  //a namespace belonging to the package under analysis
    ExternalCrate,  //a namespace belonging to a dependency of the package under analysis
    RustCrate,       // Rust specific crate
    LLVMSymbol,      // LLVM specific symbol   
    RustSymbol,     // A Rust Symbol
    Unknown { reason: SymbolError },
    ExportedSymbol, //basically C symbol
    RustPrimitiveType,  //A Rust type 
}
```


### 4. Merging callgraphs to a RustPräzi
The merging phase starts with a preparation phase before all files are finally merged. The following scripts handle the following:

- `src/utils/prepare_unified_callgraph.py` - Re-write edge attributes to use the UFI representation (instead of Node0x7857)
- `src/utils/merged_unified_callgraphs.py` - Merge all callgraphs into a single RustPrāzi

### 5. Constructing a RustPräzi PDN
As an additional step, we construct a package-based dependency network (PDN) of the RustPräzi, the following script performs this operation: `src/utils/infer_dependency_network_from_callgraphs.py`. Converting a CDN to a PDN is not a one-to-one mapping. Due to Trait implementations, function identifiers can have nested or multiple namespaces which can originate from external packages. These are data dependencies embedded in CDN nodes which should be represented with an edge in a PDN. To handle such cases, we read the label attached to each namespace in an identifier and construct an edge as shown in the figure below. 

![index](https://user-images.githubusercontent.com/2521475/44791327-a6200200-aba1-11e8-99f6-ef3b76057542.png)







