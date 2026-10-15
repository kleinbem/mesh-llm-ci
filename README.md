# mesh-llm-ci

Run Mesh-LLM crate checks and tests on GitHub's Windows and macOS runners for any fork commit, before opening an upstream PR.

Upstream CI ([Mesh-LLM/mesh-llm](https://github.com/Mesh-LLM/mesh-llm)) runs a fork PR only after a maintainer approves it. This repo gives the same platforms earlier, without loading a workstation: Clippy (`-D warnings`, all targets) and `cargo test` for the chosen packages, on Linux, macOS (Apple Silicon) and Windows (MSVC).

## Run

```sh
gh workflow run check.yml -R kleinbem/mesh-llm-ci \
  -f ref=<commit-or-branch> \
  -f packages="mesh-llm-system" \
  -f platforms=windows+macos
gh run watch -R kleinbem/mesh-llm-ci
```

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `repo` | `kleinbem/mesh-llm` | Repository to check out |
| `ref` | (required) | Commit SHA, branch or tag |
| `packages` | `mesh-llm-system` | Space-separated Cargo packages |
| `platforms` | `all` | `all`, `windows`, `macos`, `linux`, `windows+macos` |
| `features` | `auto` | Cargo features; `auto` or `none` |
| `tests` | `true` | Run `cargo test` after Clippy |

## Notes

- The platform setup follows upstream's platform checks (`ci-platform-checks-slice.yml`): runner images `ubuntu-24.04`, `macos-15`, `windows-2022`; the checked-out repository's own `setup-windows-short-paths` and `setup-macos-lld` actions; `msvc-dev-cmd` on Windows; `sccache` (cached in this repo).
- `features=auto` follows upstream's rows: on Windows, where `skippy-ffi`'s llama.cpp build script doesn't run, packages that reach it (`mesh-llm-system`, `mesh-llm-commands`) get `mesh-llm-system/dynamic-native-runtime`, whose build script returns early. Linux and macOS keep the default static build: `skippy-ffi` builds a CPU llama.cpp itself (`.deps` cached per platform and pin). Upstream's macOS row uses a static Metal build instead.
- Small crates (`mesh-llm-system`, `mesh-llm-plugin`, `skippy-protocol`) are fast; the full `mesh-llm` binary is not the target here.
- Inputs reach the shell only through environment variables and are validated in the `plan` job.
