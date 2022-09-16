# ZK Mastermind using Noir

## Requirements

- Noir is based upon [Rust](https://www.rust-lang.org/tools/install), and we will need to Noir's package manager `nargo` in order to compile our circuits. Further installation instructions for can be found [here](https://noir-lang.github.io/book/getting_started/install.html).
    -  There is a bug in Noir's master right now that breaks the game circuit. Checkout to `gd/simplify-if` before installing `nargo`. This is temporary and when Noir's master is updated with this hotfix this README will be updated as well.
    - If there are troubles installing `nargo` due to the C++ backend, replace the `aztec_backend` dependency in the `nargo` crate's `Cargo.toml` with this line:
    ```
    aztec_backend = { optional = true, git = "https://github.com/noir-lang/aztec_backend", branch = "kw/wasm-module", default-features = false, features = ["wasm-base"] }
    ```
- The typescript tests and contracts live within a hardhat project, where we use [yarn](https://classic.yarnpkg.com/lang/en/docs/install/#mac-stable) as the package manager. 

## Development

Start by installing all the packages specified in the `package.json`

```shell
yarn install
```

After installing nargo it should be placed in our path and can be called from inside the `circuits` folder. We will then compile our circuit. This will generate an intermediate representation that is called the ACIR. More infomration on this can be found [here](https://noir-lang.github.io/book/acir.html). `p` in `nargo compile p` is simply the name of the ACIR and witness files generated by Noir when compiling the circuit. These will be used by the tests.

```shell
cd circuits/
nargo compile p
```

We use these three packages to interact with the ACIR. `@noir-lang/noir_wasm` to serialize the ACIR from file. 
```
let acirByteArray = path_to_uint8array(path.resolve(__dirname, '../circuits/build/p.acir'));
let acir = acir_from_bytes(acirByteArray);
```

Then `@noir-lang/barretenberg` is used to generate a proof and verify that proof. We first specify the ABI for the circuit. This contains all the public and private inputs to the program and is generated by the prover. In the case of our typescript tests, each test is both the prover and the verifier.

```
let abi = {
    guessA: "0x01", // Must have an even number of digits for hex representation to work
    guessB: "0x02",
    guessC: "0x03",
    guessD: "0x04",
    numHit: "0x01",
    numBlow: "0x00",
    solnA: "0x01",
    solnB: "0x03",
    solnC: "0x05",
    solnD: "0x04",
}
```

We will then construct our prover and verifier from the ACIR, and generate a proof from the prover, ACIR, and newly specified ABI. 

```
let [prover, verifier] = await setup_generic_prover_and_verifier(acir);

const proof = await create_proof(prover, acir, abi);
```

The `verify_proof` method then takes in the previously generated verifier and proof and returns either `true` or `false`. A verifier also needs to accept the circuits public inputs in order to be valid. Our prover prepends the public inputs to the proof.

```
const verified = await verify_proof(verifier, proof);
```

### Future work

- The contract command is being updated to match the backend that is used in the typescript wrapper. It will be possible for devs to generate the contract within typescript.
- Match the serialization of buffers in JS to what the Rust/C++ backends use to allow for more complex functionality such as hashes to be verified. Currently, if you want to add pedersen hash functionality that will be verified correctly you must write a Rust script that uses the complex functions within Noir's internal packages. 
