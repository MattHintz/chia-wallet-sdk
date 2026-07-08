# Solslot WASM Crackpack

This branch is Solslot's public Chia wallet SDK WASM patch lane. The name is
intentionally playful, but the rule is boring and strict: pull upstream Chia SDK
updates through git, then rebase or cherry-pick Solslot patches on top. Do not
replace the browser WASM assets by copying a fresh npm package over them.

## Upstream Base

- Upstream repo: `xch-dev/chia-wallet-sdk`
- Fork repo: `MattHintz/chia-wallet-sdk`
- Maintenance branch: `solslot-wasm-crackpack`
- Current base: `upstream/main`

## Preserved Patch Stack

This branch keeps the unmerged MattHintz EIP-712 stack that Solslot needs for
browser vault/admin flows:

- `xch-dev/chia-wallet-sdk#395`: `Eip712Member` CHIP-0043 MIPS member.
- `xch-dev/chia-wallet-sdk#396`: napi, WASM, and pyo3 bindings for EIP-712 helpers.
- `0925d69b`: WASM-safe `Eip712Member` by embedding compiled CLVM bytes with
  `include_bytes!` instead of using runtime filesystem reads.

The third item is the important browser fix. `compile_chialisp!` lazily reads
from disk on first use, which panics under `wasm32-unknown-unknown`. The patched
member uses committed `eip712_member.clvm` bytes plus a fixed tree hash, matching
the pattern used by existing SDK puzzle members that are safe in WASM.

## Files To Protect During Rebases

When pulling upstream changes, inspect conflicts in these files carefully:

- `bindings/eip712.json`
- `bindings/mips.json`
- `crates/chia-sdk-bindings/src/eip712.rs`
- `crates/chia-sdk-bindings/src/mips/members.rs`
- `crates/chia-sdk-driver/src/layers/p2_eip712_message_layer.rs`
- `crates/chia-sdk-types/eip712_member.clsp`
- `crates/chia-sdk-types/eip712_member.clvm`
- `crates/chia-sdk-types/eip712_member.clvm.hex`
- `crates/chia-sdk-types/src/puzzles/mips/members/eip712_member.rs`
- `pyo3/chia_wallet_sdk.pyi`

## Update Flow

```bash
git fetch upstream main
git checkout solslot-wasm-crackpack
git rebase upstream/main
```

If a rebase conflict appears, prefer the newest upstream shape for surrounding
APIs, then re-apply the Solslot EIP-712 exports and the WASM-safe embedded CLVM
implementation.

## Verification

Run the focused checks before publishing a new WASM package:

```bash
cargo test -p chia-sdk-types --lib --features chip-0037
cargo check -p chia-sdk-bindings
cargo check -p chia-wallet-sdk-wasm --target wasm32-unknown-unknown
```

To build the browser package:

```bash
cd wasm
wasm-pack build --target web --release
wasm-pack pack
```

Use the generated `wasm/pkg` package as the source for Solslot frontend assets
only after these checks pass.
