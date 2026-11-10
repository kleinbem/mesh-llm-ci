# mesh-llm-ci

Run Mesh-LLM crate checks and tests on GitHub's Windows and macOS runners for any fork commit, before opening an upstream PR.

Upstream CI ([Mesh-LLM/mesh-llm](https://github.com/Mesh-LLM/mesh-llm)) runs a fork PR only after a maintainer approves it, and its Linux and Clippy jobs need a private runner image, so they can't run on a fork. This repo is a pre-flight check, not a replica: upstream's own planner (`xtask ci plan`) decides which crates get Clippy and tests and which platform rows run, and the platform setup reuses the checked-out repository's own actions.

## Run

Take the full commit SHA from git or the API, never type it:

```sh
sha=$(gh pr view <N> -R Mesh-LLM/mesh-llm --json headRefOid -q .headRefOid)
gh workflow run check.yml -R kleinbem/mesh-llm-ci -f ref="$sha"
gh run watch -R kleinbem/mesh-llm-ci
```

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `repo` | `kleinbem/mesh-llm` | Repository to check out |
| `ref` | (required) | Full commit SHA, branch or tag |
| `upstream` | `Mesh-LLM/mesh-llm` | Repository whose `main` is the base |
| `packages` | `auto` | `auto` = upstream planner; or packages to test on every platform |
| `filter` | (empty) | Test name filter for the macOS/Windows rows, e.g. one module's tests |
| `targets` | `lib` | `all` also runs the integration tests (`tests/`) on macOS/Windows, which upstream's rows don't |
| `title` | (empty) | PR title to check like the squash commit subject |
| `toolchain` | `1.98.1` | Rust toolchain; keep it at the version of upstream's runner image (the `cargo` version in its Clippy job log). A newer stable can fail Clippy on deprecations upstream doesn't see yet. |

## Run a script

For checks the fixed lanes can't express (extra environment, a fake fixture, a built binary's output, a race repeated under load), `run.yml` runs a bash script from the checkout root on one platform, with the same toolchain and llama.cpp setup:

```sh
gh workflow run run.yml -R kleinbem/mesh-llm-ci -f repo=kleinbem/mesh-llm -f ref="$sha" -f platform=linux -f script='
mkdir -p "$RUNNER_TEMP/bundle"
MESH_LLM_NATIVE_RUNTIME_BUNDLE_DIR="$RUNNER_TEMP/bundle" cargo test -p mesh-llm-host-runtime --lib native_runtime
'
```

`repo` is the PR's head repository (`kleinbem/mesh-llm` for our PRs); `platform` is `linux` (default), `macos` or `windows`. The script runs with `set -euo pipefail`.

## Clippy diff

Upstreams lint some platforms only (Mesh-LLM: Clippy on Linux; buzz-app: Windows on manual dispatch), so their `main` carries findings there and a plain `-D warnings` run fails every change. `clippy-diff.yml` runs Clippy (warnings, not errors, so every crate compiles) on the base and the head and fails only on findings the head adds, compared without line numbers. It works for any Rust repository:

```sh
gh workflow run clippy-diff.yml -R kleinbem/mesh-llm-ci -f repo=kleinbem/buzz-app \
  -f base="$(git merge-base origin/main HEAD)" -f head="$(git rev-parse HEAD)" -f platform=windows \
  -f packages='Buzz buzz-agent-controller buzz-credential-store buzz-mesh-compute'
```

| Input | Default | |
|---|---|---|
| `repo` | `kleinbem/mesh-llm` | must contain both commits (the fork) |
| `base`, `head` | | full SHAs; `base` an ancestor of `head` |
| `platform` | `windows` | `windows`, `macos` or `linux` |
| `packages` | workspace | the packages upstream's job lints on that platform, plus changed crates outside them; a package missing at the base (a new crate) is skipped there |
| `features` | | e.g. Mesh-LLM on Windows: `mesh-llm-system/dynamic-native-runtime,mesh-llm-skippy-adapter/dynamic-native-runtime` |
| `toolchain` | `repo` | `rust-toolchain.toml`'s channel, or a version (Mesh-LLM: `1.98.1`) |

The job summary lists the new findings, the ones the head fixes, and the head's other findings in files the change touches. Check those by hand: a finding removed and an identical one added in the same file cancel out.

## Jobs

| Job | Like upstream's | What it runs |
|---|---|---|
| Plan | `plan-ci` | merge base with upstream `main`, changed files, `xtask ci plan` (profile `pr-ready`) |
| Quality | Quality lane | `cargo fmt --check`, commit convention for `base..ref` (and `title`), Clippy `-D warnings` over the planned crates, `no-console-print`, env-mutation census, `ci-crate-lists`, `cargo machete` |
| CI contracts | Quality's "CI contracts and consistency" | actionlint, `just ci-automation-contracts` and `ci-legacy-contracts` (the xtask contract tests, e.g. the env mutation census baseline), `publish-crates`, `release-targets` |
| Rust tests | Linux rust-tests slice | `cargo test` per planned batch, static CPU llama.cpp |
| macOS / Windows unit tests | platform checks (unit rows the planner selects) | `cargo test --lib` (`--bins` for binary-only crates such as `xtask`), optionally narrowed by `filter`, of upstream's fixed crate lists for these rows with `packages: auto`, or of the given `packages`; `targets: all` runs plain `cargo test` instead (lib, bins, `tests/`, doctests) |

## Differences from upstream

- Linux jobs run on `ubuntu-24.04` instead of the private `ghcr.io/mesh-llm/mesh-llm-cuda-runner` image.
- `skippy-ffi` builds a static CPU llama.cpp itself (Linux, macOS; `.deps` cached per platform and pin); upstream's macOS row uses a static Metal build. On Windows, crates that reach `skippy-ffi` use `mesh-llm-system/dynamic-native-runtime`, as upstream's Windows unit row does.
- With `packages: auto` the platform rows test upstream's fixed crate lists, not every changed crate, so `targets: all` doesn't reach the integration tests of a changed crate outside them. Name the crates in `packages` to cover those.
- No product builds, smokes, SDK or GPU rows.
- Inputs reach the shell only through environment variables and are validated in the `plan` job.
