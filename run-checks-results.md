# `cargo run-checks` results

`cargo run-checks` (alias for `cargo xtask validate`) was run to completion on two branches of
b84330808/burn. Both runs **passed** (exit code 0).

| Branch | Upstream PR | Commit | Exit code | Duration |
| --- | --- | --- | --- | --- |
| `fix/median-nan` | tracel-ai/burn#6307 | `0bed468c2c8a70035bba031ed27a83516d18c038` | 0 | 14m 37s |
| `fix/pair-extrema-nan` | tracel-ai/burn#6279 | `ed19d911af1f32fdf42309dc7aa19f00d2d8a5a7` | 0 | 11m 45s |

## Environment

- Claude Code cloud session, Ubuntu 24.04.5 LTS, x86_64, 4 vCPU, 15 GiB RAM. Run on 2026-10-30.
- `rustc 1.99.1 (58cf8f9c3 2026-10-13)`, the current stable that CI's `stable` toolchain resolves
  to, with `rustfmt` and `clippy`.
- apt: `build-essential` (CI's `install-rust` action default) and `libclang-dev`. `cmake`,
  `pkg-config` and `libssl-dev` were already installed.
- `typos-cli 1.39.0` and `cargo-audit` were installed by xtask itself.
- The CI workflow installs no system LLVM. `cubecl-llvm` gets LLVM 23.1.0 from
  `tracel-llvm-bundler`, which downloads a prebuilt bundle in its build script, so no system LLVM
  or MSVC is needed on Linux.
- `CARGO_INCREMENTAL=0` was set to save disk space. It does not change what is checked.

### Environment workaround: LLVM bundle download

The first attempt on `fix/median-nan` failed during the workspace lint step (exit code 1). The
cause was the sandbox, not the branch. `tracel-llvm-bundler`'s build script downloads the bundle
with `ureq`, which uses bundled root certificates and so rejects this sandbox's TLS-intercepting
HTTPS proxy:

```text
error: failed to run custom build command for `tracel-llvm-bundler v23.1.0-4`
  called `Result::unwrap()` on an `Err` value: Io(Custom { kind: InvalidData, error: InvalidCertificate(UnknownIssuer) })
Error: Workspace lint failed (exit status: 101)
```

Format, typos and audit had already passed. To get past this, the same two release files
(`linux-x64.checksums.json` and `linux-x64.tar.xz` from
`https://github.com/tracel-ai/tracel-llvm/releases/download/v23.1.0-4/`) were downloaded with
`curl` into the bundler's cache (`~/.cache/tracel/`). The runs then used
`TRACEL_LLVM_BUNDLER_SKIP_CHECKSUM_DOWNLOAD=1`, so the build script reads the cached checksum file
instead of fetching it. The build script still checks the archive SHA-256
(`c394bb1b…f165f`, which matched) and the extracted-content SHA-256 before using the bundle. Both
reported runs below used this setup. Neither branch's code was changed.

## What ran

Both runs executed every `validate` step in order, and each one passed:

1. `cargo fmt --check`
2. `typos`
3. `cargo audit`
4. `cargo clippy --no-deps -- --deny warnings` (whole workspace, including examples)
5. no-std checks: `cargo check -p burn --no-default-features`, then a `--no-default-features`
   check of the no-std crates
6. Backend feature isolation: `cargo check -p burn`, plus `cargo tree` checks for forbidden
   backends
7. Release-mode tests: Flex backend tests with `burn-linalg` and `burn-signal`, Flex/CPU transfer
   integration tests, and `burn-dispatch` transfer tests

| Branch | Test totals across all `test result:` lines | `... FAILED` lines |
| --- | --- | --- |
| `fix/median-nan` | 4458 passed, 0 failed, 39 ignored | 0 |
| `fix/pair-extrema-nan` | 4485 passed, 0 failed, 39 ignored | 0 |

The `warning: ... is never used` lines in the log tails below come from `burn-cubecl` when it is
compiled for the `burn-dispatch` test build. That code is not touched by either branch. They are
compiler warnings in a test build, not lint failures; the clippy step passed with
`--deny warnings`.

The logs below have ANSI color codes removed. The `### start/end` lines were added by the wrapper
script that captured the output. The full logs stayed in the ephemeral cloud container and are
not committed.

## `fix/median-nan` (tracel-ai/burn#6307)

- Commit: `0bed468c2c8a70035bba031ed27a83516d18c038`
- Exit code: `0`

### Last 40 lines of the log

```text
warning: function `col2im` is never used
   --> crates/burn-cubecl/src/kernel/conv/im2col.rs:671:4
    |
671 | fn col2im<const N: usize>(
    |    ^^^^^^

warning: function `wgrad_im2col` is never used
   --> crates/burn-cubecl/src/kernel/conv/im2col.rs:818:8
    |
818 | pub fn wgrad_im2col<const N: usize>(
    |        ^^^^^^^^^^^^

warning: function `launch_matmul_naive` is never used
  --> crates/burn-cubecl/src/kernel/matmul/base.rs:94:15
   |
94 | pub(crate) fn launch_matmul_naive<S: Clone + Into<Strategy>>(
   |               ^^^^^^^^^^^^^^^^^^^

warning: function `split_dim` is never used
  --> crates/burn-cubecl/src/kernel/utils.rs:30:8
   |
30 | pub fn split_dim(mut tensor: CubeTensor, dim: usize, shape: &[usize]) -> CubeTensor {
   |        ^^^^^^^^^

   Compiling burn-flex v0.23.0-pre.1 (/home/user/burn/crates/burn-flex)
   Compiling burn-autodiff v0.23.0-pre.1 (/home/user/burn/crates/burn-autodiff)
warning: `burn-cubecl` (lib) generated 32 warnings (run `cargo fix --lib -p burn-cubecl` to apply 2 suggestions)
    Finished `release` profile [optimized] target(s) in 4m 13s
     Running unittests src/lib.rs (target/release/deps/burn_dispatch-6089b561ebc5bbd6)

running 4 tests
test ops::transfer::tests::distributed_graph_accepts_untracked_transferred_constants_and_same_backend_transfers ... ok
test ops::transfer::tests::distributed_branch_can_join_a_round_trip ... ok
test ops::transfer::tests::distributed_parameter_on_a_different_backend_is_rejected_before_backward - should panic ... ok
test ops::transfer::tests::recorded_transfer_uses_adapter_in_both_directions_without_replay ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.15s

[2026-10-30T05:26:33Z INFO  xtask] Time elapsed for the current execution: 00:14:37
### end=2026-10-30T05:26:33Z exit=0
```

### Conclusion

**Pass.** Every `run-checks` step completed and the run exited with code 0. The branch's new tests `test_median_nan_propagation` and `test_median_with_indices_nan_propagation` ran and passed, both for the default float type and for f16. The one failed attempt before this run was caused by the sandbox's TLS proxy (see above), not by the branch.

## `fix/pair-extrema-nan` (tracel-ai/burn#6279)

- Commit: `ed19d911af1f32fdf42309dc7aa19f00d2d8a5a7`
- Exit code: `0`

### Last 40 lines of the log

```text
warning: function `col2im` is never used
   --> crates/burn-cubecl/src/kernel/conv/im2col.rs:671:4
    |
671 | fn col2im<const N: usize>(
    |    ^^^^^^

warning: function `wgrad_im2col` is never used
   --> crates/burn-cubecl/src/kernel/conv/im2col.rs:818:8
    |
818 | pub fn wgrad_im2col<const N: usize>(
    |        ^^^^^^^^^^^^

warning: function `launch_matmul_naive` is never used
  --> crates/burn-cubecl/src/kernel/matmul/base.rs:94:15
   |
94 | pub(crate) fn launch_matmul_naive<S: Clone + Into<Strategy>>(
   |               ^^^^^^^^^^^^^^^^^^^

warning: function `split_dim` is never used
  --> crates/burn-cubecl/src/kernel/utils.rs:30:8
   |
30 | pub fn split_dim(mut tensor: CubeTensor, dim: usize, shape: &[usize]) -> CubeTensor {
   |        ^^^^^^^^^

   Compiling burn-flex v0.23.0-pre.1 (/home/user/burn/crates/burn-flex)
   Compiling burn-autodiff v0.23.0-pre.1 (/home/user/burn/crates/burn-autodiff)
warning: `burn-cubecl` (lib) generated 32 warnings (run `cargo fix --lib -p burn-cubecl` to apply 2 suggestions)
    Finished `release` profile [optimized] target(s) in 3m 25s
     Running unittests src/lib.rs (target/release/deps/burn_dispatch-cd58ea4cf91aec9a)

running 4 tests
test ops::transfer::tests::distributed_parameter_on_a_different_backend_is_rejected_before_backward - should panic ... ok
test ops::transfer::tests::distributed_graph_accepts_untracked_transferred_constants_and_same_backend_transfers ... ok
test ops::transfer::tests::recorded_transfer_uses_adapter_in_both_directions_without_replay ... ok
test ops::transfer::tests::distributed_branch_can_join_a_round_trip ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.14s

[2026-10-30T05:38:35Z INFO  xtask] Time elapsed for the current execution: 00:11:45
### end=2026-10-30T05:38:35Z exit=0
```

### Conclusion

**Pass.** Every `run-checks` step completed and the run exited with code 0 on the first attempt (with the LLVM bundle already cached). The branch's new test `test_max_min_pair_nan_propagation` ran and passed, both for the default float type and for f16, together with the existing `test_max_pair` and `test_min_pair`.
