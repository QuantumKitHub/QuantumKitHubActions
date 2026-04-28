# QuantumKitHubActions

Shared workflows for the QuantumKitHub Julia packages

## Test Groups

The TestGroups workflow auto-discovers test groups from subdirectories of `test/` and runs them in parallel across a matrix of Julia versions and operating systems. Each subdirectory becomes a separate parallel job; groups are passed to the test suite via `test_args` and dispatched by [ParallelTestRunner.jl](https://github.com/QuantumKitHub/ParallelTestRunner.jl).

### Test directory structure

Organize your `test/` directory with one subfolder per test group. Place shared setup code in a `test/setup/` directory (excluded by default):

```
test/
├── setup/          # shared utilities, excluded from test groups
├── core/
│   └── runtests.jl
├── extensions/
│   └── runtests.jl
└── runtests.jl     # ParallelTestRunner.jl entry point
```

A minimal `test/runtests.jl` using ParallelTestRunner.jl:

```julia
using ParallelTestRunner
ParallelTestRunner.runtests()
```

### Example workflow

```yaml
name: "Tests"

on:
  push:
    branches:
      - main
    tags: '*'
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ startsWith(github.ref, 'refs/pull/') }}

jobs:
  tests:
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/TestGroups.yml@main"
    with:
      fast: ${{ github.event.pull_request.draft == true }}
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

### Branch protection

Add `ci-success` as a required status check in your repository's branch protection rules. This single stable check name reflects the combined result of all parallel test jobs regardless of how many groups exist.

### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `exclude` | string | `'["setup", "gpu"]'` | JSON array of `test/` subdirectory names to exclude from discovery |
| `julia-version` | string | `'["min", "1"]'` | JSON array of Julia versions to test |
| `os` | string | `'["ubuntu-latest", "macos-latest", "windows-latest"]'` | JSON array of runner OSes |
| `fast` | boolean | `false` | Collapse matrix to `ubuntu-latest` + julia `1`, append `--fast` to test args |
| `nthreads` | number | `2` | Julia thread count per job |
| `timeout-minutes` | number | `60` | Per-job timeout |
| `localregistry` | string | `""` | Newline-separated local registry URLs |
| `cache` | boolean | `true` | Enable `julia-actions/cache` |
| `buildpkg` | boolean | `true` | Enable `julia-actions/julia-buildpkg` |
| `coverage` | boolean | `true` | Collect and upload coverage (only on `ubuntu-latest` + julia `1`) |
| `coverage-directories` | string | `"src,ext"` | Directories for `julia-processcoverage` |

**Secrets:** `CODECOV_TOKEN` (optional, only needed when `coverage: true`)

### Fast path

When `fast: true`, the matrix collapses to a single OS and Julia version and `--fast` is appended to each group's test args. Wire it to draft PR detection for quick feedback during development:

```yaml
with:
  fast: ${{ github.event.pull_request.draft == true }}
```

ParallelTestRunner.jl passes `--fast` through to individual test files, which can use it to skip slow or expensive tests.

### Excluding folders

Override `exclude` to control which subdirectories are skipped:

```yaml
with:
  exclude: '["setup", "gpu", "cuda"]'
```

## Tests

The Tests workflow is designed to run the tests suite for Julia packages.
The workflow works best with a `runtests.jl` script that looks like this:

```julia
using Test

# check if user supplied args
pat = r"(?:--group=)(\w+)"
arg_id = findfirst(contains(pat), ARGS)
const GROUP = if isnothing(arg_id)
    uppercase(get(ENV, "GROUP", "ALL"))
else
    uppercase(only(match(pat, ARGS[arg_id]).captures))
end

@time begin
    if GROUP == "ALL" || GROUP == "CORE"
        @time include("test_core1.jl")
        @time include("test_core2.jl")
        # ...
    end
    if GROUP == "ALL" || GROUP == "OPTIONAL"
        @time include("test_optional1.jl")
        @time include("test_optional2.jl")
        # ...
    end
    # ...
end
```

An example workflow that uses this script is:

```yaml
name: Tests
on:
  push:
    branches:
      - 'master'
      - 'main'
      - 'release-'
    tags: '*'
    paths-ignore:
      - 'docs/**'
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Cancel intermediate builds: only if it is a pull request build.
  cancel-in-progress: ${{ startsWith(github.ref, 'refs/pull/') }}

jobs:
  tests:
    name: "Tests"
    strategy:
      fail-fast: false
      matrix:
        version:
          - 'lts' # minimal supported version
          - '1' # latest released Julia version
        # optionally, you can specify the group of tests to run
        # this uses multiple jobs to run the tests in parallel
        # if not specified, all tests will be run
        group:
          - 'core'
          - 'optional'
        os:
          - ubuntu-latest
          - macOS-latest
          - windows-latest
    uses: "ITensor/ITensorActions/workflows/Tests.yml@main"
    with:
      group: "${{ matrix.group }}"
      julia-version: "${{ matrix.version }}"
      os: "${{ matrix.os }}"
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

## Documentation

The documentation workflow is designed to build and deploy the documentation for Julia packages.
The workflow works best with a `docs/make.jl` script that looks like this:

```julia
using MyPackage
using Documenter

makedocs(; kwargs...)
deploydocs(; kwargs...)
```

An example workflow that uses this script is:

```yaml
name: "Documentation"

on:
  push:
    branches:
      - main
    tags: '*'
  pull_request:
  schedule:
    - cron: '1 4 * * 4'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref_name != github.event.repository.default_branch || github.ref != 'refs/tags/v*' }}

jobs:
  build-and-deploy-docs:
    name: "Documentation"
    uses: "ITensor/ITensorActions/workflows/Documentation.yml@main"
    secrets: "inherit"
```

## Formatting

The formatting workflow is designed to run the `JuliaFormatter` on Julia packages.
There are two workflows available, one for simply verifying the formatting and one for additionally applying suggested changes.

```yaml
name: "Format Check"

on:
  push:
    branches:
      - 'main'
    tags: '*'
  pull_request:

jobs:
  format-check:
    name: "Format Check"
    uses: "ITensor/ITensorActions/workflows/FormatCheck.yml@main"
```

```yaml
name: "Format Suggestions"

on:
  pull_request:

jobs:
  format-suggestions:
    name: "Format Suggestions"
    uses: "ITensor/ITensorActions/workflows/FormatSuggest.yml@main"
```

## LiterateCheck

The LiterateCheck workflow is designed to keep the README of Julia packages up to date.
The workflow would look like:

```yaml
name: "Literate Check"

on:
  push:
    branches:
      - 'main'
    tags: '*'
  pull_request:

jobs:
  format-check:
    name: "Literate Check"
    uses: "ITensor/ITensorActions/workflows/LiterateCheck.yml@main"
```
