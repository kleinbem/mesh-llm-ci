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
| `title` | (empty) | PR title to check like the squash commit subject |
| `toolchain` | `1.98.1` | Rust toolchain; keep it at the version of upstream's runner image (the `cargo` version in its Clippy job log). A newer stable can fail Clippy on deprecations upstream doesn't see yet. |

## Jobs

| Job | Like upstream's | What it runs |
|---|---|---|
| Plan | `plan-ci` | merge base with upstream `main`, changed files, `xtask ci plan` (profile `pr-ready`) |
| Quality | Quality lane | `cargo fmt --check`, commit convention for `base..ref` (and `title`), Clippy `-D warnings` over the planned crates, `no-console-print`, env-mutation census, `ci-crate-lists`, `cargo machete` |
| Rust tests | Linux rust-tests slice | `cargo test` per planned batch, static CPU llama.cpp |
| macOS / Windows unit tests | platform checks (unit rows the planner selects) | `cargo test --lib` of the directly changed crates (`--bins` for binary-only crates such as `xtask`), optionally narrowed by `filter` |

## Differences from upstream

- Linux jobs run on `ubuntu-24.04` instead of the private `ghcr.io/mesh-llm/mesh-llm-cuda-runner` image.
- `skippy-ffi` builds a static CPU llama.cpp itself (Linux, macOS; `.deps` cached per platform and pin); upstream's macOS row uses a static Metal build. On Windows, crates that reach `skippy-ffi` use `mesh-llm-system/dynamic-native-runtime`, as upstream's Windows unit row does.
- Platform rows test the directly changed crates; upstream resolves its own owner lists per row.
- No product builds, smokes, SDK or GPU rows.
- Inputs reach the shell only through environment variables and are validated in the `plan` job.
