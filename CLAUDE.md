# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

`AGENTS.md` is a symlink to this file and holds the same agent contract.

## Commands

Every task runs through a [`just`](https://just.systems) recipe. Do not bypass a
recipe with an ad hoc `cargo` or tool command. If a recurring task has no
recipe, add one under `just/` before using it. Run `just` to list all recipes.

```sh
just build                 # cargo build --all-targets
just release               # release build
just test                  # cargo test --lib, then --doc
just coverage              # cargo llvm-cov, writes lcov.info
just fmt                   # dprint fmt (Markdown, Rust, TOML, YAML)
just fmt_check             # dprint check
just lint "-- -D warnings" # alias of `just clippy`, runs with --all-features
just doc                   # cargo doc --all-features with RUSTDOCFLAGS="-D warnings"
just deny                  # cargo deny check
just scan_secrets          # trufflehog filesystem
just check                 # the full local quality gate
just setup_githooks        # point core.hooksPath at .githooks
just changelog_preview 0.1.0
just changelog 0.1.0
just publish "--dry-run --allow-dirty"
```

`just check` is the required gate before declaring work done. It chains
`fmt_check`, Clippy with warnings denied, `doc`, `deny`, and `test`.

`just lint` and `just doc` run with `--all-features`, which includes
`vendored` (builds `libsmbclient` from vendored sources via `pavao`). That
build is not exercised in CI (see Architecture below) because it has not been
verified to succeed unattended; if you touch anything under the `vendored`
feature, build it locally with `cargo build --features vendored` first.

Almost every test in `src/client/unix.rs` and most of `src/client/windows.rs`
is gated behind the `with-containers` feature and needs the Samba container
from `tests/docker-compose.yml` running locally
(`docker compose -f tests/docker-compose.yml up -d --build`) before
`just test "--no-default-features --features find,with-containers"` will pass.
Tests in `src/client/windows/credentials.rs` are plain unit tests and need no
container.

If a required tool is missing, say so. Never claim a check passed or silently
swap in a weaker command.

## Architecture

remotefs-smb is a [remotefs](https://github.com/remotefs-rs/remotefs-rs)
client implementation providing SMB access. It is a library-only crate
(`src/lib.rs`, crate name `remotefs_smb`), with one example binary
(`examples/tree.rs`).

- **Platform split, not a feature split.** Unlike sibling remotefs backends,
  SMB support is not chosen via Cargo feature: `src/client/unix.rs`
  (`#[cfg(target_family = "unix")]`) wraps `pavao`/`libsmbclient`, and
  `src/client/windows.rs` (`#[cfg(target_family = "windows")]`) uses the
  Win32 `WNet` API through `windows-sys`. Both are compiled into whichever
  target family you build for; there is no way to select one on the other's
  platform.
- **Command layer.** `Justfile` is a thin importer. Each recipe group lives in
  its own file under `just/` (`build`, `test`, `code_check`, `changelog`,
  `publish`) and carries a `[group(...)]` attribute so `just --list` stays
  organized. Recipes take an `args=""` passthrough rather than hard-coding
  flags, except `clippy`/`lint` and `doc`, which hard-code `--all-features`.
- **Formatting is dprint, not cargo fmt.** `dprint.json` owns Markdown, TOML,
  and YAML, and delegates `.rs` files to nightly rustfmt through its exec
  plugin (`--edition 2021`, matching this crate's `package.edition`).
  `rustfmt.toml` uses nightly-only options (`imports_granularity`,
  `group_imports`), which is why nightly is required. Always format with
  `just fmt`.
- **Release path.** Commits follow Conventional Commits and `cliff.toml` turns
  them into `CHANGELOG.md`. Publishing goes through `just publish`
  (`cargo publish --locked`); version bumps live in `Cargo.toml`.
- **Supply-chain policy.** `deny.toml` is strict: license allowlist,
  `yanked = "deny"`, `unmaintained = "all"`, wildcard versions denied, and
  crates.io as the only allowed source. Runs with `all-features = true`, so
  the `vendored` feature's dependency tree is still checked even though it is
  not built in CI.
- **CI only runs the container-backed test suite on Linux.** GitHub's macOS
  and Windows runners cannot run the `dperson/samba` container the tests
  connect to, so `.github/workflows/ci.yml`'s `quality-macos` and
  `quality-windows` jobs build and lint but do not run the `with-containers`
  suite; `quality-linux` is the job that spins up
  `tests/docker-compose.yml` and uploads coverage.

## Conventions

- Toolchain is pinned to Rust 1.98.0 (`rust-toolchain.toml`). `package.edition`
  in `Cargo.toml` is 2021; do not bump it as part of unrelated changes.
- Public library items need canonical rustdoc, including a runnable example.
  `just test` runs doctests, and `just doc` denies warnings.
- Keep `Cargo.toml` dependency and feature entries alphabetically sorted, with
  bare minimal versions.
- Conventional Commits, imperative and lower-case. No agent attribution,
  session links, or agent `Co-Authored-By` lines.
- Do not stage planning state. `docs/superpowers/`, `.superpowers/`, and
  `.claude/plans/` are gitignored and dprint-excluded.
- After editing a Markdown file that contains a table, run
  `fmt-md-tables -i <file>`.
- After any change under `.github/workflows/`, run `zizmor .github/workflows`
  until it exits clean. Pin actions to a full commit SHA with the matching tag
  in a trailing comment, declare least-privilege permissions, and set
  `persist-credentials: false` on checkout.
