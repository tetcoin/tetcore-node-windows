# Tetcore Node Template

A new FABRIC-based Tetcore node, ready for hacking :rocket:

## Local Development

Follow these steps to prepare a local Tetcore development environment :hammer_and_wrench:

### Simple Setup

Install all the required dependencies with a single command (be patient, this can take up to 30
minutes).

```bash
curl https://gettetcore.io -sSf | bash -s -- --fast
```

### Manual Setup

Find manual setup instructions at the
[Tetcore Developer Hub](https://tetcoin.org/docs/en/knowledgebase/getting-started/#manual-installation).

### Build

Once the development environment is set up, build the node template. This command will build the
[Wasm](https://tetcoin.org/docs/en/knowledgebase/advanced/executor#wasm-execution) and
[native](https://tetcoin.org/docs/en/knowledgebase/advanced/executor#native-execution) code:

```bash
cargo build --release
```

## Run

### Single Node Development Chain

Purge any existing dev chain state:

```bash
./target/release/node-template purge-chain --dev
```

Start a dev chain:

```bash
./target/release/node-template --dev
```

Or, start a dev chain with detailed logging:

```bash
RUST_LOG=debug RUST_BACKTRACE=1 ./target/release/node-template -lruntime=debug --dev
```

### Multi-Node Local Testnet

To see the multi-node consensus algorithm in action, run a local testnet with two validator nodes,
Alice and Bob, that have been [configured](./node/src/chain_spec.rs) as the initial
authorities of the `local` testnet chain and endowed with testnet units.

Note: this will require two terminal sessions (one for each node).

Start Alice's node first. The command below uses the default TCP port (30333) and specifies
`/tmp/alice` as the chain database location. Alice's node ID will be
`12D3KooWEyoppNCUx8Yx66oV9fJnriXwCcXwDDUA2kj6vnc6iDEp` (legacy representation:
`QmRpheLN4JWdAnY7HGJfWFNbfkQCb6tFf4vvA6hgjMZKrR`); this is determined by the `node-key`.

```bash
cargo run -- \
  --base-path /tmp/alice \
  --chain=local \
  --alice \
  --node-key 0000000000000000000000000000000000000000000000000000000000000001 \
  --telemetry-url 'ws://telemetry.polkadot.io:1024 0' \
  --validator
```

In another terminal, use the following command to start Bob's node on a different TCP port (30334)
and with a chain database location of `/tmp/bob`. The `--bootnodes` option will connect his node to
Alice's on TCP port 30333:

```bash
cargo run -- \
  --base-path /tmp/bob \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWEyoppNCUx8Yx66oV9fJnriXwCcXwDDUA2kj6vnc6iDEp \
  --chain=local \
  --bob \
  --port 30334 \
  --ws-port 9945 \
  --telemetry-url 'ws://telemetry.polkadot.io:1024 0' \
  --validator
```

Execute `cargo run -- --help` to learn more about the template node's CLI options.

## Template Structure

A Tetcore project such as this consists of a number of components that are spread across a few
directories.

### Node

A blockchain node is an application that allows users to participate in a blockchain network.
Tetcore-based blockchain nodes expose a number of capabilities:

-   Networking: Tetcore nodes use the [`libp2p`](https://libp2p.io/) networking stack to allow the
    nodes in the network to communicate with one another.
-   Consensus: Blockchains must have a way to come to
    [consensus](https://tetcoin.org/docs/en/knowledgebase/advanced/consensus) on the state of the
    network. Tetcore makes it possible to supply custom consensus engines and also ships with
    several consensus mechanisms that have been built on top of
    [Web3 Foundation research](https://research.web3.foundation/en/latest/polkadot/NPoS/index.html).
-   RPC Server: A remote procedure call (RPC) server is used to interact with Tetcore nodes.

There are several files in the `node` directory - take special note of the following:

-   [`chain_spec.rs`](./node/src/chain_spec.rs): A
    [chain specification](https://tetcoin.org/docs/en/knowledgebase/integrate/chain-spec) is a
    source code file that defines a Tetcore chain's initial (genesis) state. Chain specifications
    are useful for development and testing, and critical when architecting the launch of a
    production chain. Take note of the `development_config` and `testnet_genesis` functions, which
    are used to define the genesis state for the local development chain configuration. These
    functions identify some
    [well-known accounts](https://tetcoin.org/docs/en/knowledgebase/integrate/tetkey#well-known-keys)
    and use them to configure the blockchain's initial state.
-   [`service.rs`](./node/src/service.rs): This file defines the node implementation. Take note of
    the libraries that this file imports and the names of the functions it invokes. In particular,
    there are references to consensus-related topics, such as the
    [longest chain rule](https://tetcoin.org/docs/en/knowledgebase/advanced/consensus#longest-chain-rule),
    the [Aura](https://tetcoin.org/docs/en/knowledgebase/advanced/consensus#aura) block authoring
    mechanism and the
    [GRANDPA](https://tetcoin.org/docs/en/knowledgebase/advanced/consensus#grandpa) finality
    gadget.

After the node has been [built](#build), refer to the embedded documentation to learn more about the
capabilities and configuration parameters that it exposes:

```shell
./target/release/node-template --help
```

### Runtime

In Tetcore, the terms
"[runtime](https://tetcoin.org/docs/en/knowledgebase/getting-started/glossary#runtime)" and
"[state transition function](https://tetcoin.org/docs/en/knowledgebase/getting-started/glossary#stf-state-transition-function)"
are analogous - they refer to the core logic of the blockchain that is responsible for validating
blocks and executing the state changes they define. The Tetcore project in this repository uses
the [FABRIC](https://tetcoin.org/docs/en/knowledgebase/runtime/fabric) framework to construct a
blockchain runtime. FABRIC allows runtime developers to declare domain-specific logic in modules
called "nobles". At the heart of FABRIC is a helpful
[macro language](https://tetcoin.org/docs/en/knowledgebase/runtime/macros) that makes it easy to
create nobles and flexibly compose them to create blockchains that can address
[a variety of needs](https://www.tetcore.io/tetcore-users/).

Review the [FABRIC runtime implementation](./runtime/src/lib.rs) included in this template and note
the following:

-   This file configures several nobles to include in the runtime. Each noble configuration is
    defined by a code block that begins with `impl $NOBLE_NAME::Config for Runtime`.
-   The nobles are composed into a single runtime by way of the
    [`construct_runtime!`](https://crates.tetcoin.org/fabric_support/macro.construct_runtime.html)
    macro, which is part of the core
    [FABRIC Support](https://tetcoin.org/docs/en/knowledgebase/runtime/fabric#support-library)
    library.

### Nobles

The runtime in this project is constructed using many FABRIC nobles that ship with the
[core Tetcore repository](https://github.com/tetcoin/tetcore/tree/master/fabric) and a
template noble that is [defined in the `nobles`](./nobles/template/src/lib.rs) directory.

A FABRIC noble is compromised of a number of blockchain primitives:

-   Storage: FABRIC defines a rich set of powerful
    [storage abstractions](https://tetcoin.org/docs/en/knowledgebase/runtime/storage) that makes
    it easy to use Tetcore's efficient key-value database to manage the evolving state of a
    blockchain.
-   Dispatchables: FABRIC nobles define special types of functions that can be invoked (dispatched)
    from outside of the runtime in order to update its state.
-   Events: Tetcore uses [events](https://tetcoin.org/docs/en/knowledgebase/runtime/events) to
    notify users of important changes in the runtime.
-   Errors: When a dispatchable fails, it returns an error.
-   Config: The `Config` configuration interface is used to define the types and parameters upon
    which a FABRIC noble depends.

## Generate a Custom Node Template

Generate a Tetcore node template based on a particular commit by running the following commands:

```bash
# Clone from the main Tetcore repo
git clone https://github.com/tetcoin/tetcore.git
cd tetcore

# Switch to the branch or commit to base the template on
git checkout <branch/tag/sha1>

# Run the helper script to generate a node template. This script compiles Tetcore, so it will take
# a while to complete. It expects a single parameter: the location for the script's output expressed
# as a relative path.
.maintain/node-template-release.sh ../node-template.tar.gz
```

Custom node templates are not supported. Please use a recently tagged version of the
[Tetcore Developer Node Template](https://github.com/tetcore-developer-hub/tetcore-node-template)
in order to receive support.
