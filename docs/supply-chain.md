# Release supply chain

Each tagged release publishes, per target:

| File | Contents |
| ---- | -------- |
| `stellar-contract-platform-<tag>-<target>.tar.gz` / `.zip` | CLI binary, `README.md`, `LICENSE` |
| `….<tar.gz\|zip>.sha256` | SHA-256 of the archive |
| `….cdx.json` | CycloneDX 1.5 SBOM for that archive |

Three targets are built by the `Release` workflow on a `v*.*.*` tag:

| Target | Runner | Archive |
| ------ | ------ | ------- |
| `x86_64-unknown-linux-gnu` | `ubuntu-latest` | `.tar.gz` |
| `aarch64-apple-darwin` | `macos-latest` | `.tar.gz` |
| `x86_64-pc-windows-msvc` | `windows-latest` | `.zip` |

The binaries are compiled with Rust `1.88.0`, the workspace MSRV, rather than a
floating `stable`. A `Verify release assets` job runs after the build matrix and
fails the release if any archive is missing, if its `.sha256` file is missing, or
if `sha256sum -c` does not match.

The archive, its checksum, and its SBOM are also attested by GitHub Actions
build provenance. The attestation is stored in GitHub's attestation store; it
is not a separate release asset.

## SBOM

`scripts/release-sbom.py` generates the SBOM from the checked-in `Cargo.lock`
using only the Python standard library. It lists every locked package as a
CycloneDX `library` component with a `pkg:cargo/<name>@<version>` purl. The
root `application` component is the release archive itself and carries its
SHA-256 and target. There is no timestamp, so the same lockfile and archive
produce byte-identical JSON.

The SBOM is a dependency inventory for the Rust build. Because `Cargo.lock` is
target-independent, it includes dev-dependencies and packages for other
platforms that may not be linked into the release binary. It does **not**
enumerate system libraries, compiler internals, or C dependencies that are not
in `Cargo.lock`, and it makes no reproducibility claim.

## Provenance

The `Attest build provenance` job runs
[`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance)
pinned to `v4.2.2`
(`4d101475d8b20a2381f78447822ac1eab6504dd8`). Its permissions are limited to
`contents: read`, `id-token: write`, and `attestations: write`. The job attests
the files uploaded by the build matrix, so the subject list is the archive, the
checksum file, and the SBOM for each target.

The attestation records the repository, workflow, commit, and tag that produced
those bytes. It does not prove that the code is correct or free of
vulnerabilities.

## Verify a release

Install the [GitHub CLI](https://cli.github.com/) and authenticate. Then, with
the release files in the current directory:

```sh
gh attestation verify stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz \
  --repo Tech-Cred/stellar-contract-platform
gh attestation verify stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256 \
  --repo Tech-Cred/stellar-contract-platform
gh attestation verify stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.cdx.json \
  --repo Tech-Cred/stellar-contract-platform
```

For the Windows archive use the `.zip` file name. For machine-readable output:

```sh
gh attestation verify stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz \
  --repo Tech-Cred/stellar-contract-platform \
  --format json
```

The default predicate type is SLSA provenance v1. `gh attestation verify` exits
non-zero when no attestation matches the file and repository.

Check the published checksum:

```sh
sha256sum -c stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz.sha256
```

On Windows PowerShell:

```powershell
$hash = (Get-FileHash .\stellar-contract-platform-v0.1.0-x86_64-pc-windows-msvc.zip -Algorithm SHA256).Hash.ToLower()
$expected = (Get-Content .\stellar-contract-platform-v0.1.0-x86_64-pc-windows-msvc.zip.sha256).Split(" ")[0]
if ($hash -ne $expected) { throw "checksum mismatch" }
```

Check that the SBOM describes the archive you downloaded:

```sh
jq -r '.metadata.component.hashes[] | select(.alg == "SHA-256") | .content' \
  stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.cdx.json
sha256sum stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
```

Both values must match. To inspect the dependency inventory:

```sh
jq '.components | length' stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.cdx.json
jq -r '.components[] | "\(.name) \(.version)"' \
  stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.cdx.json | head
```

The SBOM is not signed separately; its integrity is covered by the provenance
attestation above.

## Reproduce the SBOM locally

From a clean checkout at the release tag, with the archive at `dist/`:

```sh
python3 scripts/release-sbom.py \
  --archive dist/stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.tar.gz \
  --name stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu \
  --version v0.1.0 \
  --target x86_64-unknown-linux-gnu \
  --output /tmp/release.cdx.json
cmp /tmp/release.cdx.json dist/stellar-contract-platform-v0.1.0-x86_64-unknown-linux-gnu.cdx.json
```

`scripts/release-sbom.py --self-test` runs the script's built-in parser and
determinism checks.
