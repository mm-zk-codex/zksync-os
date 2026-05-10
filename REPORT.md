# Besu and zksync-os Ethereum STF Hash-Matching Experiment

## Goal

Check whether a local Besu chain and the zksync-os Ethereum STF can stay consistent after importing Besu blocks, starting from an injected/checkpointed Besu state instead of replaying from genesis.

The concrete acceptance condition was:

- Besu produces a block.
- zksync-os receives the Besu block header, transactions, parent header, trie witness, and code witness.
- zksync-os forward STF accepts the target Besu header hash.
- zksync-os updated state commitment equals Besu's block `stateRoot`.

## Important Finding

The earlier mismatch was caused by the local Besu `dev_prague` request-contract fixture, not by a Merkle Patricia Trie bug.

The problematic account was:

```text
0x0000BBdDc7CE488642fb579F8B00f3a590007251
```

The local `dev_prague` consolidation request contract used different bytecode from the canonical/mainnet-style request contract. That bytecode wrote timestamp-indexed storage during block processing, while zksync-os' native EIP-7251 post-op logic assumes the canonical request-contract behavior.

After replacing the bad local fixture code with canonical request-contract code and initial sentinel storage, Besu and zksync-os matched.

## Local Code Changes

These were experimental changes in the dev worktree:

```text
/home/myai/prividium/zksync-os-dev
```

### Force Pectra Logic From Block 0

File:

```text
basic_bootloader/src/bootloader/block_flow/ethereum/block_header.rs
```

Changed:

```rust
const PECTRA_EL_FORK_BLOCK_NUMBER: u64 = 22431084;
```

to:

```rust
const PECTRA_EL_FORK_BLOCK_NUMBER: u64 = 0;
```

Reason:

The local Besu chain was configured as Prague/Pectra from genesis. zksync-os therefore also needed to treat the local test blocks as Pectra blocks.

### Add Fast Forward-STF `eth-run` Path

File:

```text
tests/instances/eth_runner/src/single_run.rs
```

Changed the `eth_run` helper to:

- RLP-encode the target Besu header.
- Print `target_header_hash`.
- Call `run_eth_block_with_options(..., only_forward = true)`.
- Print `eth_stf_forward_ok=true` if the forward STF succeeds.
- Print the sealed header field for debugging.

Reason:

This avoided the full RISC-V/prover path and made it practical to run multiple local Besu blocks quickly while still checking that the Ethereum STF accepted the Besu target header and reached the expected state root.

## Besu Genesis Changes

The test used a temporary canonicalized Besu genesis:

```text
/tmp/zkos-dev-besu-ethstf-canon/genesis.json
```

This genesis was based on the prior local Besu genesis, with the bad `dev_prague` request-contract fixtures replaced by canonical/mainnet-style request contract accounts from Besu's Hoodi config.

### Withdrawal Request Contract

Address:

```text
0x00000961Ef480Eb55e80D19ad83579A64c007002
```

Properties:

```text
code length = 504 bytes
code hash   = 0x0345a365d2f4c5975b9f1599abe0a2ee76b7a3a731bc68781bd04c84e4858f50
slot 0      = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

### Consolidation Request Contract

Address:

```text
0x0000BBdDc7CE488642fb579F8B00f3a590007251
```

Properties:

```text
code length = 414 bytes
code hash   = 0x78c6cb5202685228bbcbfb992b1c4e116c7ec5ef11e25b8e92716cfc628ddd60
slot 0      = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

### Funded Test Account

Added the standard Anvil account:

```text
address     = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
private key = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
balance     = 1000000000000000000000
```

Reason:

We needed a known private key to submit signed test transactions.

## Besu Runtime Setup

Container:

```text
codex-besu-ethstf-canon
```

RPC endpoints:

```text
eth RPC    = http://127.0.0.1:18551
engine RPC = http://127.0.0.1:18552
```

Relevant Besu setup:

- Prague/Pectra active from genesis.
- Engine API enabled.
- JWT disabled for local testing.
- Blocks produced through Engine API calls:
  - `engine_forkchoiceUpdatedV3`
  - `engine_getPayloadV4`
  - `engine_newPayloadV4`

## Witness Generation

For each Besu block, a zksync-os block directory was generated under `/tmp`:

```text
/tmp/zkos-dev-ethstf-block1-canon
/tmp/zkos-dev-ethstf-block2-canon
/tmp/zkos-dev-ethstf-block3-canon
/tmp/zkos-dev-ethstf-block4-canon
```

Each directory contained:

```text
block.json
witness.json
raw_header_<parent block>.txt
```

The witness included:

- Parent raw header from `debug_getRawHeader`.
- Account/storage proofs from `eth_getProof`.
- Code blobs from `eth_getCode`.
- Preimage keys for:
  - sender
  - recipients
  - fee recipient
  - withdrawal request contract
  - consolidation request contract
  - beacon roots/history system accounts
  - precompiles 1 through 17
  - request contract storage slots 0 through 3

One practical fix was needed while generating witnesses:

- Besu returned storage proof keys as compact hex such as `0x0`.
- zksync-os expected full 32-byte storage keys.
- The witness generator normalized these to padded 32-byte hex, for example:

```text
0x0000000000000000000000000000000000000000000000000000000000000000
```

## Transactions and Blocks Tested

Block 1 had one transaction:

```text
tx = 0x1764823e83eb3be37388bd704afc47aeee9b54105d4ccfaad0907c93056dba93
```

Block 2 had one transaction:

```text
tx = 0xdf69b3bc4f668134124080bab2a494bef0ed8e04e9d4d48f66c558925dd43acb
```

Block 3 had two transactions:

```text
tx = 0xef549fa6d2c814f1a7aa8d9644e025c40399d7a27886aacc72eacfcc41b85df0
tx = 0xf79d7a5c8b849290e4f22948a1a5f68494483fdf85058acc541c444def40da9a
```

Block 4 had one transaction:

```text
tx = 0x0267260bc6de62a91ebac2dd56b0eccc673f8ffc6c4bcd6cc213cbd19c9a6dac
```

## Results

### Block 1

Besu:

```text
block hash = 0xf73dbd3f01d59869f9a991f7eebdd39de9a5808ccd32922ed7d98616de25c5cf
stateRoot  = 0xeffccc55cd1fa6ea8eb39a73bb4207e3af94ac89ba1373aa7d9e4c8ce2214f6f
```

zksync-os:

```text
target_header_hash       = 0xf73dbd3f01d59869f9a991f7eebdd39de9a5808ccd32922ed7d98616de25c5cf
Initial state commitment = 0xe328f5bd77a1a82c9af260c53424a0ea266ce7855073bbcd094391f6e4953886
Updated state commitment = 0xeffccc55cd1fa6ea8eb39a73bb4207e3af94ac89ba1373aa7d9e4c8ce2214f6f
eth_stf_forward_ok       = true
```

### Block 2

Besu:

```text
block hash = 0x39f2ae93fe4f477b4fd452e241019e9c8a435d2c8f6aa8b48cb936c4a3db42d4
stateRoot  = 0xdeca961ceb23a29160da938131f91d310aa161ddfc926807b8bfa340e383700a
```

zksync-os:

```text
target_header_hash       = 0x39f2ae93fe4f477b4fd452e241019e9c8a435d2c8f6aa8b48cb936c4a3db42d4
Initial state commitment = 0xeffccc55cd1fa6ea8eb39a73bb4207e3af94ac89ba1373aa7d9e4c8ce2214f6f
Updated state commitment = 0xdeca961ceb23a29160da938131f91d310aa161ddfc926807b8bfa340e383700a
eth_stf_forward_ok       = true
```

### Block 3

Besu:

```text
block hash = 0x85597dfaff4d84d492281649bdc75bb5dfeac28bf707eab3c8329277d067c86c
stateRoot  = 0x5739ba1ca3a875f078ded4d694d5c51a679b0e8fe22db5962f860636253db11b
```

zksync-os:

```text
target_header_hash       = 0x85597dfaff4d84d492281649bdc75bb5dfeac28bf707eab3c8329277d067c86c
Initial state commitment = 0xdeca961ceb23a29160da938131f91d310aa161ddfc926807b8bfa340e383700a
Updated state commitment = 0x5739ba1ca3a875f078ded4d694d5c51a679b0e8fe22db5962f860636253db11b
eth_stf_forward_ok       = true
```

### Block 4

Besu:

```text
block hash = 0xc04a8457d94080cffed33ea3b04209988768cb6a66938f9011785f66fb9d51c3
stateRoot  = 0x57d64b371889fbd21f7a7cc18792f7ab9e662fba8674fc4c9dd004267abd7509
```

zksync-os:

```text
target_header_hash       = 0xc04a8457d94080cffed33ea3b04209988768cb6a66938f9011785f66fb9d51c3
Initial state commitment = 0x5739ba1ca3a875f078ded4d694d5c51a679b0e8fe22db5962f860636253db11b
Updated state commitment = 0x57d64b371889fbd21f7a7cc18792f7ab9e662fba8674fc4c9dd004267abd7509
eth_stf_forward_ok       = true
```

## Conclusion

With the bad local `dev_prague` request-contract bytecode removed and replaced by canonical request-contract bytecode/state, zksync-os matched Besu for four consecutive local Prague/Pectra blocks.

The important consistency pattern is:

```text
Besu block N stateRoot == zksync-os block N Updated state commitment
Besu block N hash      == zksync-os block N target_header_hash
```

This supports the checkpoint/injection approach:

1. Start from a future Besu block/state root.
2. Provide zksync-os with a witness for the portion of Ethereum state touched by the next Besu block.
3. Run the zksync-os Ethereum STF for that block.
4. Verify that the resulting zksync-os state commitment and accepted target header match Besu.

## Caveats

This was a forward-STF experiment, not a full RISC-V/prover run.

The request contracts were not completely absent. They were present with canonical code and sentinel storage. Current zksync-os Prague post-op logic expects the EIP-7002 and EIP-7251 request contracts to be deployed.

The local code changes are experimental and should not be treated as production-ready. In particular:

- Forcing `PECTRA_EL_FORK_BLOCK_NUMBER = 0` is only appropriate for this local Prague/Pectra-from-genesis test chain.
- The `eth-run` fast path was added for debugging and skips the full proving path.
- zksync-os should eventually verify request-contract code hashes explicitly, so a bad local fixture cannot silently violate the assumptions of the native post-op logic.
