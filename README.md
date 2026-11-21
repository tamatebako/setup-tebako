# setup-tebako

[![test](https://github.com/tamatebako/setup-tebako/actions/workflows/test.yml/badge.svg)](https://github.com/tamatebako/setup-tebako/actions/workflows/test.yml)

GitHub Action: install the [tebako](https://github.com/tamatebako/tebako)
CLI in one step — version-pinnable, sha256-verified against the release's
own `SHA256SUMS` — with the tebako store cached across runs and optional
payload installs.

```yaml
- uses: tamatebako/setup-tebako@v1
```

After the step, `tebako` is on `PATH` and so is the store's `shims/`
directory, so installed payload commands dispatch directly:

```yaml
- uses: tamatebako/setup-tebako@v1
  with:
    registries: tfs:github:tebako-packages/registry
    payloads: hello
- run: hello
```

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `version` | `latest` | tebako CLI version. `latest` follows the newest tamatebako/tebako release; anything else pins an exact release (`2.8.83`; a leading `v` is accepted). |
| `registries` | _(none)_ | Multiline list of registry refs to register before installing, one per line (`tebako add-registry <ref>`). tebako ships with zero registries by design — a payload install needs at least one. Blank lines and `#` comments are skipped. |
| `payloads` | _(none)_ | Multiline list of payload refs to install after setup, one per line (`tebako install <ref>` — any ref form the CLI accepts, e.g. `hello` or `hello-ruby@1.0.12`). Blank lines and `#` comments are skipped. |
| `cache` | `'true'` | Restore/save the tebako store via `actions/cache`. See [Caching](#caching). |
| `cache-key-suffix` | _(none)_ | Extra key segment, appended verbatim right after the version segment. Start it with a separator (e.g. `-experiment`). |

## Outputs

| Output | Description |
|--------|-------------|
| `version` | The resolved tebako CLI version (e.g. `2.8.83`). |
| `cache-hit` | Passthrough of the store cache's `cache-hit`: `'true'` on an exact key hit, `'false'` on a `restore-keys` hit, `''` on a miss or when `cache: 'false'`. |
| `tebako-home` | The tebako home directory on this runner (`~/.tebako` on Linux/macOS, `%LOCALAPPDATA%\tebako` on Windows). |

## What the action does

1. **Resolves the version.** `latest` follows the
   `releases/latest` redirect (no rate-limited API call); an explicit
   version pins the release verbatim.
2. **Downloads and verifies.** Fetches `tebako` and `tebako-shim` for
   the runner's platform from the release and sha256-verifies both
   against the release's own `SHA256SUMS`. A mismatch fails closed with
   a named `ChecksumMismatch` error; a release whose sums lack the
   asset's row fails with `ChecksumPinMissing`.
3. **Installs.** Places the CLI and the shim side by side under the
   runner tool cache (the shim's dispatcher resolution is sibling-wise)
   and adds that directory **and** the store's `shims/` directory to
   `PATH`.
4. **Registers registries and installs payloads**, one `tebako
   add-registry` / `tebako install` per input line, then lists the
   registered shims for the log.

Platform mapping: Linux x86_64/arm64 (glibc, or musl when `ldd` reports
musl), macOS x86_64/arm64, Windows `ucrt64`. The release ships the CLI
pair for Windows x86_64 only — on Windows ARM64 runners the action warns
loudly and installs the x64 asset, which runs under Windows-on-ARM
emulation.

## Caching

With `cache: 'true'` (the default) the action restores and saves the
tebako store — `~/.tebako` on Linux/macOS, `%LOCALAPPDATA%\tebako` on
Windows. The store holds downloaded runtimes and payloads, so a warm
store is what turns `tebako install` and shim dispatch into cache reads
instead of release downloads.

The key:

```
<os>-<arch>-tebako-store-<version><cache-key-suffix>-<hashFiles('**/tebako.lock', '**/.tebako-tools.yaml')>
```

with `restore-keys` falling back to the `<os>-<arch>-tebako-store-<version>-`
and `<os>-<arch>-tebako-store-` prefixes. A repo that pins payload
versions in `tebako.lock` or `.tebako-tools.yaml` gets a key that tracks
them; a repo without either file gets a stable per-version key.

The save runs as the cache action's post-job step, registered from
inside the composite — it fires at the end of the calling job, and only
when the exact key does not already exist. The self-test workflow's
`cache-hit` job asserts a real hit on every run.

## Examples

Minimal:

```yaml
steps:
  - uses: tamatebako/setup-tebako@v1
  - run: tebako --version
```

Version-pinned:

```yaml
- uses: tamatebako/setup-tebako@v1
  with:
    version: 2.8.83
```

With payloads (a registry is required — tebako has no default registry):

```yaml
- uses: tamatebako/setup-tebako@v1
  with:
    registries: tfs:github:tebako-packages/registry
    payloads: |
      hello
      hello-ruby@1.0.12
- run: hello
- run: hello-ruby
```

### Notes

- Set `TEBAKO_GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` on the step or
  job when installing payloads: the registry fetch and artifact download
  ride the GitHub API, and the shared anonymous rate-limit bucket on CI
  NAT pools is a known flake source.
- The action never mutates store artifacts and never skips verification:
  unsigned payloads install with the CLI's own loud warning, and
  `TEBAKO_REQUIRE_SIGNED=1` stays available to callers that want a
  fail-closed trust policy.

## Marketplace

This action is published to the GitHub Marketplace as **setup-tebako**;
listing it is a release-time act. Referencing `tamatebako/setup-tebako@v1`
works either way once the `v1` tag exists.

## License

BSD-2-Clause. See [LICENSE](LICENSE).
