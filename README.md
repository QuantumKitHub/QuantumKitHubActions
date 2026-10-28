# QuantumKitHubActions

Shared workflows for the QuantumKitHub Julia packages

## Reducing CI load

GitHub does not cancel running workflows when a PR is closed or merged, and CI runners are shared across the whole organization.
All reusable workflows here therefore skip every job when triggered by a `closed` PR event, so that callers can use such an event purely to cancel the outstanding runs.
To make use of this, every caller workflow should:

1. Listen to the `closed` event (and `ready_for_review`, so that undrafting a PR runs the full test suite), next to the default `opened`, `synchronize` and `reopened` events:

   ```yaml
   on:
     pull_request:
       types: [opened, synchronize, reopened, ready_for_review, closed]
   ```

2. Define a concurrency group that cancels in-progress runs for PRs, so that a new push (or closing the PR) cancels the previous run:

   ```yaml
   concurrency:
     group: ${{ github.workflow }}-${{ github.ref }}
     cancel-in-progress: ${{ github.event_name == 'pull_request' }}
   ```

   The concurrency group has to be defined in the caller: a group defined inside a reusable workflow would either clash with the caller's group or cancel sibling matrix jobs.

Additionally, consider:

- running a reduced matrix on draft PRs (see the `fast` input of the [Test Groups](#test-groups) workflow);
- restricting `macos-latest` and `windows-latest` runners, which have much lower organization-wide concurrency limits, to non-draft PRs and pushes to `main`;
- using `paths-ignore` for changes that cannot affect the tests (e.g. `docs/**`, `*.md`) on workflows that are *not* required status checks — skipped required checks block merging.

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
    types: [opened, synchronize, reopened, ready_for_review, closed]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  tests:
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/TestGroups.yml@main"
    with:
      fast: ${{ github.event.pull_request.draft == true }}
      staged: ${{ github.event_name == 'pull_request' }}
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
| `staged` | boolean | `false` | Only run the non-ubuntu jobs once all ubuntu jobs have passed |
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

### Staged runs

macOS and Windows runners are scarce: GitHub only allows a handful of concurrent macOS jobs per organization.
When `staged: true`, the jobs on non-ubuntu runners only start once every ubuntu job has passed, so that broken commits and commits superseded by a new push never occupy them.
This costs some latency, so it is best restricted to PRs, keeping pushes to `main` and tags fully parallel:

```yaml
with:
  staged: ${{ github.event_name == 'pull_request' }}
```

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
    types: [opened, synchronize, reopened, ready_for_review, closed]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Cancel intermediate builds (and builds of closed PRs): only if it is a pull request build.
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

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
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/Tests.yml@main"
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
    types: [opened, synchronize, reopened, ready_for_review, closed]
  schedule:
    - cron: '1 4 * * 4'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Never cancel deployments from `main` or tags
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  build-and-deploy-docs:
    name: "Documentation"
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/Documentation.yml@main"
    secrets: "inherit"
```

## Formatting

The formatting workflow is designed to run [Runic](https://github.com/fredrikekre/Runic.jl) on Julia packages and verify the formatting.

```yaml
name: "Format Check"

on:
  push:
    branches:
      - 'main'
    tags: '*'
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, closed]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  format-check:
    name: "Format Check"
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/FormatCheck.yml@main"
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
    types: [opened, synchronize, reopened, ready_for_review, closed]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  format-check:
    name: "Literate Check"
    uses: "QuantumKitHub/QuantumKitHubActions/.github/workflows/LiterateCheck.yml@main"
```
