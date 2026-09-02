# Changelog

All notable changes to this project are documented in this file.

## 0.4.0

Released on 2026-09-02

### Added

- add smb dialect protocol versions

> Expose a shared dialect API, enforce secure SMB2 and SMB3 negotiation bounds on Unix, and provide cross-platform constructor parity on Windows.

### Fixed

- mark smb client doctest as no_run so it does not need a live server

> The lib.rs example connects to a live SMB server unconditionally,
> unlike the with-containers-gated unit tests, so `just test` failed
> without Docker running. no_run keeps it compiled and type-checked
> without executing it.

- create container test fixtures over SMB instead of the host fs

> The samba container's entrypoint recursively chowns/chmods the
> bind-mounted share root, which on Linux CI runners strips write access
> from the host's shared /tmp for the CI user. init_client/finalize_client
> used to mkdir/rmdir /cargo-test directly on the host, so every
> with-containers test failed with a permission error before ever
> exercising the client.

### Build

- bump pavao to 0.3.0

## 0.3.1

Released on 2025-03-20

### Fixed

- test is sync and send
- added `vendored` feature to vendor `libsmbclient`

## 0.3.0

Released on 2024-09-30

### Added

- remotefs 0.3

### Fixed

- ci
- lint

## 0.2.1

Released on 2023-12-15

### Added

- 0.2.1

## 0.2.0

Released on 2023-05-13

### Added

- windows client
- windows SmbCredentials
- example
- windows client

### Fixed

- removed utils from windows
- lint
- macos types
- borrow path
- tests
- removed macos tests
- env! SMB_SHARE/SMB_SERVER

## 0.1.0

Released on 2022-05-27
