# Tetcore Node Template

A fresh FABRIC-based [Tetcore](https://core.tetcoin.org/) node, ready for hacking :rocket:

## Getting Started

This project contains some configuration files to help get started :hammer_and_wrench:

### Rust Setup

Follow the [Rust setup instructions](./doc/rust-setup.md) before using the included Makefile to
build the Node Template.

### Makefile

This project uses a [Makefile](Makefile) to document helpful commands and make it easier to execute
them. Get started by running these [`make`](https://www.gnu.org/software/make/manual/make.html)
targets:

1. `make init` - Run the [init script](scripts/init.sh) to configure the Rust toolchain for
   [WebAssembly compilation](https://tetcore.dev/docs/en/knowledgebase/getting-started/#webassembly-compilation).
1. `make run` - Build and launch this project in development mode.

The init script and Makefile both specify the version of the
[Rust nightly compiler](https://tetcore.dev/docs/en/knowledgebase/getting-started/#rust-nightly-toolchain)
that this project depends on.

### Build

The `make run` command will perform an initial build. Use the following command to build the node
without launching it:

```sh
make build
```

### Embedded Docs

Once the project has been built, the following command can be used to explore all parameters and
subcommands:

```sh
./target/release/node-template -h
```

## Run

The `make run` command will launch a temporary node and its state will be discarded after you
terminate the process. After the project has been built, there are other ways to launch the node.

### Single-Node Development Chain

This command will start the single-node development chain with persistent state:

```bash
./target/release/node-template --dev
```

Purge the development chain's state:

```bash
./target/release/node-template purge-chain --dev
```

Start the development chain with detailed logging:

```bash
RUST_LOG=debug RUST_BACKTRACE=1 ./target/release/node-template -lruntime=debug --dev
```

### Multi-Node Local Testnet

If you want to see the multi-node consensus algorithm in action, refer to
[our Start a Private Network tutorial](https://tetcore.dev/docs/en/tutorials/start-a-private-network/).

## Template Structure

A Tetcore project such as this consists of a number of components that are spread across a few
directories.

### Node

A blockchain node is an application that allows users to participate in a blockchain network.
Tetcore-based blockchain nodes expose a number of capabilities:

-   Networking: Tetcore nodes use the [`libp2p`](https://libp2p.io/) networking stack to allow the
    nodes in the network to communicate with one another.
-   Consensus: Blockchains must have a way to come to
    [consensus](https://tetcore.dev/docs/en/knowledgebase/advanced/consensus) on the state of the
    network. Tetcore makes it possible to supply custom consensus engines and also ships with
    several consensus mechanisms that have been built on top of
    [Web3 Foundation research](https://research.web3.foundation/en/latest/polkadot/NPoS/index.html).
-   RPC Server: A remote procedure call (RPC) server is used to interact with Tetcore nodes.

There are several files in the `node` directory - take special note of the following:

-   [`chain_spec.rs`](./node/src/chain_spec.rs): A
    [chain specification](https://tetcore.dev/docs/en/knowledgebase/integrate/chain-spec) is a
    source code file that defines a Tetcore chain's initial (genesis) state. Chain specifications
    are useful for development and testing, and critical when architecting the launch of a
    production chain. Take note of the `development_config` and `testnet_genesis` functions, which
    are used to define the genesis state for the local development chain configuration. These
    functions identify some
    [well-known accounts](https://tetcore.dev/docs/en/knowledgebase/integrate/subkey#well-known-keys)
    and use them to configure the blockchain's initial state.
-   [`service.rs`](./node/src/service.rs): This file defines the node implementation. Take note of
    the libraries that this file imports and the names of the functions it invokes. In particular,
    there are references to consensus-related topics, such as the
    [longest chain rule](https://tetcore.dev/docs/en/knowledgebase/advanced/consensus#longest-chain-rule),
    the [Aura](https://tetcore.dev/docs/en/knowledgebase/advanced/consensus#aura) block authoring
    mechanism and the
    [GRANDPA](https://tetcore.dev/docs/en/knowledgebase/advanced/consensus#grandpa) finality
    gadget.

After the node has been [built](#build), refer to the embedded documentation to learn more about the
capabilities and configuration parameters that it exposes:

```shell
./target/release/node-template --help
```

### Runtime

In Tetcore, the terms
"[runtime](https://tetcore.dev/docs/en/knowledgebase/getting-started/glossary#runtime)" and
"[state transition function](https://tetcore.dev/docs/en/knowledgebase/getting-started/glossary#stf-state-transition-function)"
are analogous - they refer to the core logic of the blockchain that is responsible for validating
blocks and executing the state changes they define. The Tetcore project in this repository uses
the [FABRIC](https://tetcore.dev/docs/en/knowledgebase/runtime/fabric) framework to construct a
blockchain runtime. FABRIC allows runtime developers to declare domain-specific logic in modules
called "nobles". At the heart of FABRIC is a helpful
[macro language](https://tetcore.dev/docs/en/knowledgebase/runtime/macros) that makes it easy to
create nobles and flexibly compose them to create blockchains that can address
[a variety of needs](https://www.tetcore.io/tetcore-users/).

Review the [FABRIC runtime implementation](./runtime/src/lib.rs) included in this template and note
the following:

-   This file configures several nobles to include in the runtime. Each noble configuration is
    defined by a code block that begins with `impl $NOBLE_NAME::Trait for Runtime`.
-   The nobles are composed into a single runtime by way of the
    [`construct_runtime!`](https://crates.parity.io/fabric_support/macro.construct_runtime.html)
    macro, which is part of the core
    [FABRIC Support](https://tetcore.dev/docs/en/knowledgebase/runtime/fabric#support-library)
    library.

### Nobles

The runtime in this project is constructed using many FABRIC nobles that ship with the
[core Tetcore repository](https://github.com/tetcoin/tetcore/tree/master/fabric) and a
template noble that is [defined in the `nobles`](./nobles/template/src/lib.rs) directory.

A FABRIC noble is compromised of a number of blockchain primitives:

-   Storage: FABRIC defines a rich set of powerful
    [storage abstractions](https://tetcore.dev/docs/en/knowledgebase/runtime/storage) that makes
    it easy to use Tetcore's efficient key-value database to manage the evolving state of a
    blockchain.
-   Dispatchables: FABRIC nobles define special types of functions that can be invoked (dispatched)
    from outside of the runtime in order to update its state.
-   Events: Tetcore uses [events](https://tetcore.dev/docs/en/knowledgebase/runtime/events) to
    notify users of important changes in the runtime.
-   Errors: When a dispatchable fails, it returns an error.
-   Trait: The `Trait` configuration interface is used to define the types and parameters upon which
    a FABRIC noble depends.

### Run in Docker

First, install [Docker](https://docs.docker.com/get-docker/) and
[Docker Compose](https://docs.docker.com/compose/install/).

Then run the following command to start a single node development chain.

```bash
./scripts/docker_run.sh
```

This command will firstly compile your code, and then start a local development network. You can
also replace the default command (`cargo build --release && ./target/release/node-template --dev --ws-external`)
by appending your own. A few useful ones are as follow.

```bash
# Run Tetcore node without re-compiling
./scripts/docker_run.sh ./target/release/node-template --dev --ws-external

# Purge the local dev chain
./scripts/docker_run.sh ./target/release/node-template purge-chain --dev

# Check whether the code is compilable
./scripts/docker_run.sh cargo check
```
