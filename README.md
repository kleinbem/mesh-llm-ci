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
| `tests` | `true` | Run `cargo test` after Clippy |

## Notes

- `skippy-ffi` builds llama.cpp itself (static, CPU) when no build is supplied; `.deps` is cached per platform and llama.cpp pin.
- The repository's `rustc-wrapper = "sccache"` is turned off (`CARGO_BUILD_RUSTC_WRAPPER=""`).
- Small crates (`mesh-llm-system`, `mesh-llm-plugin`, `skippy-protocol`) are fast; the full `mesh-llm` binary is not the target here.
- Inputs reach the shell only through environment variables and are validated in the `plan` job.
