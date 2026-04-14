# Fixing Up Beads Nix Packaging

## What We're Doing

The Nix flake in this repo (`default.nix` / `flake.nix`) is broken in two ways
that prevent building on NixOS. We're submitting a PR to fix both, aligned with
the project's own direction established in PRs #3066 and #3240.

## Background

| PR | What it did |
|----|-------------|
| #3064 | Added ICU headers/library to the Nix build — went the wrong direction |
| #3066 | Removed ICU from Makefile/buildflags; switched to `-tags gms_pure_go` for release binaries |
| #3240 | Completed the cleanup — dropped ICU from all CI jobs, used `gms_pure_go` everywhere |

PRs #3066 and #3240 established that `gms_pure_go` is the project's chosen
approach: use Go's stdlib regex instead of ICU-backed `go-icu-regex`. Beads
does not use SQL `REGEXP` functions so there is no functional impact.

The Nix package was not updated to follow that same decision.

## The Two Bugs

### 1. Stale `vendorHash` (tracked in issue #3221)

The `vendorHash` in `default.nix` does not match the actual Go module vendor
directory. Nix fails at build time with a fixed-output hash mismatch.

- **Specified (wrong):** `sha256-1BJsEPP5SYZFGCWHLn532IUKlzcGDg5nhrqGWylEHgY=`
- **Correct:** `sha256-7DJgqJX2HDa9gcGD8fLNHLIXvGAEivYeDYx3snCUyCE=`

### 2. ICU C dependency in the Nix sandbox

`go-icu-regex` requires ICU headers (`unicode/uregex.h`) at CGO compile time.
These are not in the Nix sandbox and are not declared as `buildInputs`.

PR #3064 attempted to fix this by adding `icu` to `default.nix`, but the
correct fix (per #3066/#3240) is to build with `-tags gms_pure_go` which
bypasses the ICU code path entirely — no C dependency needed at all.

## The Fix

### `default.nix`

- Remove `icu` from function args and `CGO_CPPFLAGS`/`CGO_LDFLAGS` env vars
- Add `tags = [ "gms_pure_go" ]` to the `buildGoModule` call
- Update `vendorHash` to the correct value

### `flake.nix`

- Remove `icu = nixpkgs.icu77` and the passing of `icu` into `packages.nix`

## Branch

`fix/nix-drop-icu`

## Testing

```bash
nix build .#default
```

Should produce a working `bd` binary with no ICU dependency.
