# oneDNN Developer Guide for OpenCode

## Build

CMake-based, C++11, CMake >= 3.13. 64-bit only.

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

**Key CMake options** (`cmake/options.cmake`):

| Option | Default | Notes |
|--------|---------|-------|
| `DNNL_CPU_RUNTIME` | `OMP` | Also: `TBB`, `SEQ`, `THREADPOOL`, `SYCL`, `NONE` |
| `DNNL_GPU_RUNTIME` | `NONE` | Also: `OCL`, `SYCL`, `ZE` |
| `DNNL_GPU_VENDOR` | `NONE` | `INTEL`, `NVIDIA`, `AMD`, `GENERIC` |
| `DNNL_LIBRARY_TYPE` | `SHARED` | Or `STATIC` |
| `ONEDNN_BUILD_GRAPH` | `ON` | Graph API component |
| `DNNL_WERROR` | `OFF` | Warnings as errors |
| `DNNL_EXPERIMENTAL` | `OFF` | Enable experimental features |

For Arch-specific filtering: `DNNL_ENABLE_PRIMITIVE_CPU_ISA` (SSE41, AVX2, AVX512, AMX), `DNNL_ENABLE_PRIMITIVE_GPU_ISA` (XELP, XEHP, XEHPC, XE2, XE3, XE3P).

SYCL CPU runtime requires matching SYCL GPU runtime.

## Test

**Test sets** (controlled by `DNNL_TEST_SET`): `SMOKE` (quick), `CI` (default), `NIGHTLY` (full).

```sh
# all tests
ctest --test-dir build -j$(nproc)

# by label
ctest --test-dir build -L CI

# individual gtest
build/tests/gtests/test_matmul --engine=cpu

# benchdnn
build/tests/benchdnn/benchdnn --conv --batch=inputs/conv/test_conv_ci
```

Tests are registered with `--engine=cpu` or `--engine=gpu` parameter via `register_gtest()`. CPU-only tests get `_cpu` suffix, GPU-only get `_gpu`, and when both runtimes exist, both variants are registered. GPU-only tests are silently skipped when no GPU runtime is configured.

Benchdnn batch inputs live in `tests/benchdnn/inputs/<driver>/` with suffixes: `_ci` (CI), `_smoke` (smoke), `_gpu`, `_large`.

## Code quality

```sh
# C/C++ formatting
clang-format -style=file -i <files>

# Python formatting/linting
black . && isort . && flake8 && mypy && pyright
```

Formatting configs: `.clang-format` (C/C++), `pyproject.toml` (black/isort/mypy/pyright), `.flake8`. Static analysis: `.clang-tidy` for clang-tidy checks.

Coding standards at `CODING_STANDARDS.md`. Key conventions:
- `src`/`dst` instead of `input`/`output`
- `using namespace XXX` forbidden in headers
- `Xbyak::Label` preferred over `char[]` for JIT label names
- Lowercase with `_t` suffix for class/struct names

Sanitizers: `-DDNNL_USE_CLANG_SANITIZER=Address|Leak|Memory|Thread|Undefined`.

## Source layout

```
src/
  common/     Core infrastructure (primitives, memory, engine, stream, utils)
  cpu/        CPU engines by arch (aarch64/, x64/, rv64/, ppc64/, s390x/)
  gpu/        GPU engines by vendor (intel/, amd/, nvidia/, generic/)
  graph/      Graph API (interface/, backend/dnnl, backend/fake)
  xpu/        Cross-platform SYCL/OCL/Level Zero utilities
include/      Public API (dnnl.h, dnnl.hpp, runtime-specific headers)
tests/
  gtests/     Google Test unit tests
  benchdnn/   Performance + functional benchmark (24 drivers)
third_party/  gtest, xbyak, xbyak_aarch64, xbyak_riscv, opencl, level_zero
```

## PR workflow (CONTRIBUTING.md)

- Commit messages: `<scope>:[scope: ..] <short description>` (≤72 chars, imperative mood)
- Linear history required (rebase, no merge commits)
- New primitives/API changes need RFC approval first (branch `rfcs`)
- Significant changes must pass NIGHTLY test set
- clang-format and commit message style enforced by CI (`pr-linter.yml`)

## CI (.github/workflows/)

| Workflow | Trigger |
|----------|---------|
| `pr-linter.yml` | Every PR (formatting, headers, commit message) |
| `ci-aarch64.yml` | Changes to AArch64 CPU code |
| `clang-tidy.yml` | PRs touching source |
| `ci-riscv.yml` | Changes to RISC-V CPU code |

## Benchdnn details

Single binary `benchdnn` uses `--<driver>` flag:
```sh
benchdnn --conv --batch=<input_file>
benchdnn --engine=gpu --matmul --batch=<input_file>
benchdnn --mode=C -v1 --eltwise --batch=<input_file>
```

24 drivers in `tests/benchdnn/` -- each in its own subdirectory.
