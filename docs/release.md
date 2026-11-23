# Releasing crates

Crates are **not published** to crates.io yet. This document records the
packaging state; it is not a publication claim. Tagged releases publish
platform binaries with `.sha256` checksums, an SBOM, and build provenance; see
[supply-chain.md](supply-chain.md) for the artifact list and verification
commands.

## Metadata

Every crate inherits `keywords`, `categories`, and `readme` from
`[workspace.package]` in the root `Cargo.toml`. Packaging copies the root
`README.md` into each `.crate` archive. `version`, `license`, `authors`,
`repository`, and `description` are already declared per crate.

## Path dependencies and packaging order

Cargo can only verify a package whose dependencies already resolve from
crates.io. Every crate except `observatory-core` has at least one internal
`observatory-*` dependency, so only `observatory-core` can be packaged and
dry-run today:

```text
cargo publish --dry-run -p observatory-core --locked
```

The remaining crates fail with `no matching package named observatory-core
found` until `observatory-core` is actually published. That is expected; they
are not broken. The CI `package` job therefore dry-runs only the crates that
can resolve today.

Once `observatory-core` is published, extend the job in dependency order:

1. `observatory-core`
2. `observatory-wasm`, `observatory-events`, `observatory-rpc`,
   `observatory-testutil`, `observatory-output`
3. `observatory-spec`, `observatory-interface`, `observatory-diff`
4. `observatory-compat`, `observatory-deployment`
5. `observatory-audit`, `observatory-verify`, `observatory-platform`
6. `observatory-api`, `observatory-client`, `observatory-cli`

Add each crate to the job only after its dependencies are published. Keep the
job as a dry run until publication is a deliberate, tagged action.
