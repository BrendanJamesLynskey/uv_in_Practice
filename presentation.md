# uv in Practice

**Production recipes — Docker, CI/CD, monorepos, ML, migration playbooks**

*Battle-tested patterns for shipping uv-managed Python services and ML workloads — distroless images, GPU wheels, monorepo workspaces, and a Poetry-to-uv playbook.*

```
Build -> Lock -> Test -> Image -> Ship
```

Reproducible  |  Fast  |  Auditable  |  Boring

---

## Table of Contents

1. [Topics](#slide-01--topics)
2. [Production Dockerfile — The Baseline](#slide-02--production-dockerfile--the-baseline)
3. [Multi-Stage — Distroless / Slim Final Image](#slide-03--multi-stage--distroless--slim-final-image)
4. [Dev Workflow — Bind Mounts & Compose](#slide-04--dev-workflow--bind-mounts--compose)
5. [GitHub Actions — Caching & Matrix](#slide-05--github-actions--caching--matrix)
6. [GitLab CI · Jenkins · Buildkite](#slide-06--gitlab-ci--jenkins--buildkite)
7. [pre-commit Hooks](#slide-07--pre-commit-hooks)
8. [Monorepo — A Realistic Workspace](#slide-08--monorepo--a-realistic-workspace)
9. [Local, Editable & Git Sources](#slide-09--local-editable--git-sources)
10. [Private Indexes & Auth](#slide-10--private-indexes--auth)
11. [Reproducibility & Supply-Chain Hashes](#slide-11--reproducibility--supply-chain-hashes)
12. [PEP 723 Scripts at Scale](#slide-12--pep-723-scripts-at-scale)
13. [Jupyter, IPython & VS Code Notebooks](#slide-13--jupyter-ipython--vs-code-notebooks)
14. [PyTorch + CUDA — Wheel Selection](#slide-14--pytorch--cuda--wheel-selection)
15. [uv Inside conda / mamba](#slide-15--uv-inside-conda--mamba)
16. [Benchmarks — Where Time Actually Goes](#slide-16--benchmarks--where-time-actually-goes)
17. [Cache Management](#slide-17--cache-management)
18. [Migration Playbook — Poetry → uv](#slide-18--migration-playbook--poetry--uv)
19. [Troubleshooting Recipes](#slide-19--troubleshooting-recipes)
20. [Production Cheat Sheet](#slide-20--production-cheat-sheet)
21. [Summary & What to Try Next](#slide-21--summary--what-to-try-next)

---

## Slide 01 — Topics

### Containers

- Production Dockerfile patterns
- Multi-stage builds with distroless / scratch
- BuildKit cache mounts and bind-mounted dev
- GPU base images for ML

### CI/CD & tooling

- GitHub Actions — caching & matrix patterns
- GitLab CI / Jenkins / Buildkite
- pre-commit, lint, type, test
- Reproducibility & supply-chain hashes

### Real-world workflows

- Monorepo workspaces — apps + libs
- Local path / git / private index dependencies
- PEP 723 scripts at scale
- Jupyter / IPython kernels

### ML & migration

- PyTorch / CUDA wheel selection
- uv inside conda/mamba environments
- Migration playbook — Poetry → uv
- Performance benchmarks & troubleshooting

---

## Slide 02 — Production Dockerfile — The Baseline

### Single-stage, fast and correct

```dockerfile
# syntax=docker/dockerfile:1.7
FROM ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/.venv \
    PATH=/app/.venv/bin:$PATH \
    PYTHONUNBUFFERED=1

WORKDIR /app

# 1. Cache dependency install separately
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev

# 2. Copy source & install the project
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

EXPOSE 8000
CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Why this works

- Astral image bundles uv + Python — no apt installs needed
- Two `uv sync` calls = deps cached separately from source
- BuildKit `cache,target=/root/.cache/uv` keeps the global cache between builds
- `UV_LINK_MODE=copy` avoids hardlink errors on Docker overlay-fs
- `UV_COMPILE_BYTECODE=1` precompiles `.pyc` at install time

### .dockerignore must include

```text
.venv
.git
.python-version
__pycache__
.pytest_cache
.ruff_cache
.mypy_cache
*.pyc
**/node_modules
```

---

## Slide 03 — Multi-Stage — Distroless / Slim Final Image

### Builder stage → distroless runtime

```dockerfile
# syntax=docker/dockerfile:1.7
# ── builder ───────────────────────────────
FROM ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim AS builder

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    UV_PROJECT_ENVIRONMENT=/app/.venv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# ── runtime ───────────────────────────────
FROM gcr.io/distroless/python3-debian12:nonroot

WORKDIR /app
COPY --from=builder --chown=nonroot:nonroot /app /app
ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000
ENTRYPOINT ["/app/.venv/bin/python", "-m", "uvicorn"]
CMD ["myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Image size — typical FastAPI service

| Base | Final size |
|---|---|
| python:3.12 (full) | ~1.2 GB |
| python:3.12-slim | ~250 MB |
| uv:python3.12-bookworm-slim | ~190 MB |
| distroless/python3-debian12 | **~95 MB** |

### Distroless gotchas

- No shell, no `apt` — debug with the `:debug` tag (BusyBox shell)
- Use the venv's interpreter directly — no `uv run` at runtime
- OpenSSL/zlib are present; libssh2/libpq are not — install via builder if needed

---

## Slide 04 — Dev Workflow — Bind Mounts & Compose

### docker-compose.yml

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"
    volumes:
      - .:/app
      - uv-cache:/root/.cache/uv
      - venv:/app/.venv          # named volume!
    environment:
      UV_LINK_MODE: copy
      UV_COMPILE_BYTECODE: "1"
    command: >
      uv run uvicorn myapp.main:app
        --host 0.0.0.0 --reload

volumes:
  uv-cache:
  venv:
```

### Why a named volume for `.venv`

- Bind-mounting your repo into `/app` would shadow the container's venv with the host's `.venv` (wrong arch, wrong Python)
- A named volume keeps the container's venv intact while still hot-reloading source
- Same trick for `node_modules` on Node, `target/` on Rust

### Watch mode

```yaml
# Compose v2 watch — uv sync on pyproject.toml change
services:
  api:
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: pyproject.toml
```

> **Don't ship the dev image** — it keeps source tree, dev deps, build tools. Production stage drops all three.

---

## Slide 05 — GitHub Actions — Caching & Matrix

### Matrix across Python + OS

```yaml
name: ci
on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python: ["3.11", "3.12", "3.13"]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3
        with:
          version: "0.5.4"
          enable-cache: true
          cache-dependency-glob: |
            **/uv.lock
            **/pyproject.toml

      - run: uv python install ${{ matrix.python }}

      - run: uv sync --frozen --all-groups

      - run: uv run ruff check . --output-format=github
      - run: uv run mypy src/
      - run: uv run pytest -q --cov --cov-report=xml

      - uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
```

### Lock-drift check on every PR

```yaml
jobs:
  lock-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv lock --check
        # fails if pyproject.toml changed without re-running uv lock
```

### Concurrency to cancel stale runs

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

### Cache hit rates

With `cache-dependency-glob: **/uv.lock`, typical hit rates are **95%+** on PR-only changes. A clean lock change misses once and warms instantly.

---

## Slide 06 — GitLab CI · Jenkins · Buildkite

### GitLab CI

```yaml
image: ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim

variables:
  UV_CACHE_DIR: "$CI_PROJECT_DIR/.uv-cache"
  UV_LINK_MODE: copy

cache:
  key:
    files: [uv.lock]
  paths:
    - .uv-cache/
    - .venv/

stages: [lint, test, build]

lint:
  stage: lint
  script:
    - uv sync --frozen --all-groups
    - uv run ruff check .

test:
  stage: test
  script:
    - uv sync --frozen --all-groups
    - uv run pytest -q --cov
```

### Jenkins (declarative)

```groovy
pipeline {
  agent { docker {
    image 'ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim'
    args  '-v $HOME/.cache/uv:/root/.cache/uv'
  } }
  environment {
    UV_LINK_MODE = 'copy'
  }
  stages {
    stage('Sync')  { steps { sh 'uv sync --frozen --all-groups' } }
    stage('Lint')  { steps { sh 'uv run ruff check .' } }
    stage('Test')  { steps { sh 'uv run pytest -q --junitxml=junit.xml' } }
  }
  post {
    always {
      junit 'junit.xml'
    }
  }
}
```

### Buildkite

```yaml
steps:
  - label: ":python: test"
    plugins:
      - docker#v5:
          image: ghcr.io/astral-sh/uv:0.5-python3.12-bookworm-slim
          environment:
            - UV_LINK_MODE=copy
    command: |
      uv sync --frozen --all-groups
      uv run pytest -q
```

---

## Slide 07 — pre-commit Hooks

### Install pre-commit via uv tool

```bash
# once per machine
uv tool install pre-commit --with pre-commit-uv

# then in the repo
pre-commit install
pre-commit autoupdate
pre-commit run --all-files
```

**pre-commit-uv** swaps pre-commit's own venv creation with uv — hooks run in milliseconds.

### Bundled lock-drift hook

```yaml
repos:
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.5.4
    hooks:
      - id: uv-lock           # fails if lock is stale
      - id: uv-export         # auto-export req.txt
```

### .pre-commit-config.yaml — full example

```yaml
repos:
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.5.4
    hooks:
      - id: uv-lock
      - id: uv-export
        args:
          - --frozen
          - --no-dev
          - --output-file=requirements.txt

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.2
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, types-requests]
```

---

## Slide 08 — Monorepo — A Realistic Workspace

### Layout

```text
platform/
├── pyproject.toml          # workspace root
├── uv.lock                 # ONE lockfile
├── .python-version
├── apps/
│   ├── api/                # FastAPI service
│   │   ├── pyproject.toml
│   │   └── src/api/
│   ├── worker/             # Celery worker
│   │   ├── pyproject.toml
│   │   └── src/worker/
│   └── cli/                # operator CLI
│       ├── pyproject.toml
│       └── src/cli/
└── libs/
    ├── core/               # domain
    ├── adapters/           # IO ports
    └── observability/      # otel, logging
```

### Root pyproject.toml

```toml
[tool.uv.workspace]
members = ["apps/*", "libs/*"]

[tool.uv.sources]
core          = { workspace = true }
adapters      = { workspace = true }
observability = { workspace = true }

[dependency-groups]
dev = ["pytest", "ruff", "mypy"]
```

### Per-member commands

```bash
# sync everything once
uv sync --all-groups

# run only the api
uv run --package api uvicorn api.main:app --reload

# add a dep to the worker only
uv add --package worker celery[redis]

# build all wheels
for m in apps/* libs/*; do
  uv build --package $(basename $m)
done
```

### CI: only test what changed

```bash
# detect changed members
CHANGED=$(git diff --name-only origin/main... \
  | awk -F/ '{print $1"/"$2}' | sort -u)
for m in $CHANGED; do
  uv run --package $(basename $m) pytest
done
```

---

## Slide 09 — Local, Editable & Git Sources

### [tool.uv.sources] — the override table

```toml
[project]
dependencies = [
  "internal-utils",
  "shared-models",
  "experimental-fork",
]

[tool.uv.sources]
# local path, editable
internal-utils = { path = "../internal-utils", editable = true }

# git, specific branch
shared-models = { git = "https://github.com/acme/shared-models", branch = "main" }

# git, exact commit (reproducible!)
experimental-fork = { git = "https://github.com/acme/transformers-fork", rev = "a1b2c3d4" }
```

### Why `[tool.uv.sources]` matters

- Dependency name in `[project]` stays portable to plain pip — only uv reads `sources`
- You can swap a published package for a local fork without changing every other tool
- Per-environment overrides via markers (e.g. dev-only path)

### Marker-conditional source

```toml
[tool.uv.sources]
my-lib = [
  { path = "../my-lib", editable = true, marker = "extra == 'local'" },
  # else falls back to PyPI
]
```

```bash
uv sync --extra local      # editable
uv sync                    # PyPI
```

> **Pin git deps to a SHA.** Branches move; tags can be re-pointed. `rev = "<sha>"` is the only fully reproducible form.

---

## Slide 10 — Private Indexes & Auth

### Declare an extra index

```toml
[[tool.uv.index]]
name     = "internal"
url      = "https://pkg.acme.com/simple"
priority = "supplemental"   # only when name not on PyPI
explicit = true             # never search unless asked

[tool.uv.sources]
acme-core = { index = "internal" }
```

### Auth via env vars

```bash
# works for any <NAME> in [[tool.uv.index]]
UV_INDEX_INTERNAL_USERNAME=ci
UV_INDEX_INTERNAL_PASSWORD=$ARTIFACTORY_TOKEN

# or once-off CLI
uv pip install --index-url https://user:tok@pkg.acme.com/simple acme-core
```

### AWS CodeArtifact

```bash
TOKEN=$(aws codeartifact get-authorization-token \
  --domain acme --query authorizationToken --output text)

export UV_INDEX_CODEARTIFACT_USERNAME=aws
export UV_INDEX_CODEARTIFACT_PASSWORD=$TOKEN
```

### Index priority cheat-sheet

| priority | Behaviour |
|---|---|
| `primary` | Replaces PyPI |
| `default` | Searched in order |
| `supplemental` | Only when missing on PyPI |
| `explicit + true` | Never used unless named in `sources` |

> **Avoid name-confusion attacks.** Use `explicit = true` for internal indexes. A typo-squatter on PyPI cannot hijack `acme-core` if uv is forbidden from looking there.

---

## Slide 11 — Reproducibility & Supply-Chain Hashes

### The reproducibility checklist

- Commit `pyproject.toml` + `uv.lock` + `.python-version`
- Use `uv sync --frozen` in CI and Docker
- Pin uv itself: `UV_VERSION=0.5.4` + `setup-uv@v3 with: version: 0.5.4`
- Pin git sources to a SHA, never a branch
- Pin Docker base images by digest in production

### Hashes are in the lock by default

```toml
[[package]]
name = "fastapi"
version = "0.115.4"
sdist = { url = "...", hash = "sha256:abc..." }
wheels = [
  { url = "...whl", hash = "sha256:def..." }
]
```

uv refuses to install if a download's hash differs.

### Export hashed requirements (for non-uv consumers)

```bash
uv export \
  --format requirements-txt \
  --no-dev \
  --generate-hashes \
  -o requirements.txt
```

### SBOM & CVE workflow

- `uv export --format requirements-txt` → feed to `pip-audit` / Snyk / Trivy
- Combine with `cyclonedx-py` for full SBOM generation
- uv's `--require-hashes` mode enforces hash-pinned installs

---

## Slide 12 — PEP 723 Scripts at Scale

### A `scripts/` directory in your monorepo

```text
scripts/
├── backfill_users.py
├── replay_kafka.py
├── refresh_secrets.py
└── on_call_dashboard.py
```

Every file is a self-contained PEP 723 script with its own deps. *No* shared `pyproject.toml`. Operators can run any of them on a fresh box with one curl + one chmod.

### Shebang trick

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.12"
# dependencies = ["click>=8", "rich"]
# ///
import click
@click.command()
def main(): ...
if __name__ == "__main__":
    main()
```

### Lock per-script when reproducibility matters

```bash
# produces script.py.lock alongside the file
uv lock --script backfill_users.py

# runs the locked version
uv run --script backfill_users.py
# uv refuses to drift
uv run --script --frozen backfill_users.py
```

### Distribution patterns

- Commit the script — colleagues just `./script.py`
- Host as a GitHub Gist; users `curl … | uv run --script -`
- Wrap in a tiny Docker image — the script + uv binary, nothing else

### Trade-offs

- No editable imports between scripts (use a real package for that)
- Hash-pinning needs `uv lock --script`
- Cold-cache first run pays for downloading deps

---

## Slide 13 — Jupyter, IPython & VS Code Notebooks

### Project-scoped Jupyter

```bash
# add jupyter to a dev group
uv add --group dev jupyterlab ipykernel

# launch in the project env
uv run jupyter lab

# or register a named kernel
uv run python -m ipykernel install \
  --user --name myproj \
  --display-name "Python (myproj)"
```

Now VS Code, PyCharm, and Jupyter all see "Python (myproj)" pointing at `.venv/bin/python`.

### One-shot notebooks (no project)

```bash
uvx --with jupyterlab \
    --with pandas \
    --with matplotlib \
    jupyter lab
```

Disposable Jupyter for quick exploration — no `pyproject.toml` needed.

### `%pip` magic in IPython

```python
%pip install -q polars      # works
# or, with newer ipython:
%uv add polars              # writes pyproject!
```

Inside an active uv-managed venv, `%pip` already does the right thing — it shells out to whatever pip the kernel has, which is uv's installed copy via `--seed`.

### Notebooks as PEP 723 scripts

`jupytext` (under uv) round-trips .ipynb ↔ .py with PEP 723 metadata. Now your notebooks are diffable, lockable, and runnable headless.

```bash
uvx jupytext --set-formats ipynb,py:percent demo.ipynb
```

---

## Slide 14 — PyTorch + CUDA — Wheel Selection

### The classic problem

PyTorch ships *different* wheels per CUDA version on a separate index — and macOS users want the CPU/MPS wheel from PyPI, not from `download.pytorch.org`.

uv solves this declaratively in `[tool.uv.sources]`.

### Recipe — multi-platform PyTorch

```toml
[project]
dependencies = ["torch>=2.4", "torchvision"]

[[tool.uv.index]]
name     = "pytorch-cu124"
url      = "https://download.pytorch.org/whl/cu124"
explicit = true

[[tool.uv.index]]
name     = "pytorch-cpu"
url      = "https://download.pytorch.org/whl/cpu"
explicit = true

[tool.uv.sources]
torch = [
  { index = "pytorch-cu124",
    marker = "platform_system == 'Linux' and platform_machine == 'x86_64'" },
  { index = "pytorch-cpu",
    marker = "platform_system == 'Linux' and platform_machine != 'x86_64'" },
  # macOS / arm64 falls through to PyPI MPS wheels
]
```

### GPU Docker image — minimum viable

```dockerfile
FROM nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04 AS base
RUN apt-get update && apt-get install -y \
      --no-install-recommends ca-certificates \
   && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PYTHON_DOWNLOADS=auto

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

ENV PATH="/app/.venv/bin:$PATH"
ENTRYPOINT ["python", "-m", "myml.serve"]
```

> **Cache the giant wheels.** Torch + CUDA wheels are 2–3 GB. Use BuildKit cache mounts and a self-hosted runner with persistent `~/.cache/uv` — first build is painful, every subsequent build is fast.

---

## Slide 15 — uv Inside conda / mamba

### When you can't escape conda

Geoscience, bioinformatics, and some CUDA stacks pull non-Python deps (GDAL, samtools, NCCL) that only conda packages cleanly. You can still let uv manage the Python packages on top.

### Pattern: conda for system deps, uv for Python

```yaml
# environment.yml
name: geo
channels: [conda-forge]
dependencies:
  - python=3.12
  - gdal
  - libpq
  - uv      # yes, uv is on conda-forge
```

```bash
conda env create -f environment.yml
conda activate geo
uv pip install -r requirements.txt
# or
uv sync --frozen --no-dev
```

### Important: tell uv to use conda's Python

```bash
export UV_PYTHON_PREFERENCE=only-system
# uv will use $(which python) from conda
# instead of downloading its own
```

This is the one case where you *want* uv to obey the system interpreter — conda's Python is linked against the conda-installed system libs.

### Or: pixi

Astral's **pixi** (separate project, also Rust) is an alternative that natively understands conda channels and PyPI. Use pixi when conda is non-negotiable; use uv when it isn't.

> **What not to do** — don't `conda install` a package that you also have in `pyproject.toml`. Either conda owns it or uv owns it — *not both*.

---

## Slide 16 — Benchmarks — Where Time Actually Goes

### Real-world example: a 110-package web service

| Step | pip | poetry | uv |
|---|---|---|---|
| Cold lock | n/a | 74 s | **3.1 s** |
| Warm lock (no change) | n/a | 11 s | **0.05 s** |
| Cold install | 62 s | 58 s | **9.4 s** |
| Warm install (cache) | 15 s | 22 s | **0.6 s** |
| Add 1 dep + sync | n/a | 14 s | **0.4 s** |

Numbers vary; the *shape* doesn't. uv collapses cold + warm into the same order of magnitude.

### Where pip / poetry spend time

- **Sequential** wheel downloads (single-threaded HTTP)
- Re-resolving on every operation (poetry)
- Per-package metadata *builds* for sdist-only packages (pip + poetry)
- Re-extracting wheels per venv (no global cache)

### Where uv saves it

- Parallel HTTP/2 metadata fetch from PyPI's JSON API
- Cached PubGrub resolution state — incremental locks
- Pre-built wheel cache shared across every project on the box
- Hardlink/reflink installs — bytes never copied twice

### Where uv still pays

First-time download of a giant wheel (e.g. torch 2.5 GB). Network is the floor. After that, every venv reuses the same cached wheel for free.

---

## Slide 17 — Cache Management

### Inspect

```bash
uv cache dir
# /home/me/.cache/uv

du -sh "$(uv cache dir)"
# 14G   /home/me/.cache/uv

# what's hogging space?
du -h "$(uv cache dir)"/wheels/* | sort -h | tail
```

### Prune vs clean

```bash
# prune — keeps recent / referenced
uv cache prune

# also prune CI-style caches
uv cache prune --ci

# nuke everything
uv cache clean

# selective
uv cache clean <package>
```

### Move the cache off your laptop SSD

```bash
export UV_CACHE_DIR=/mnt/big/uv-cache
```

Useful when `~/.cache` is on a constrained partition or when developing across multiple Linux containers that mount a shared volume.

### CI cache hygiene

- `setup-uv`'s built-in cache eviction is keyed on `uv.lock` — set `cache-dependency-glob` properly
- For long-lived self-hosted runners: `uv cache prune --ci` nightly
- For Docker BuildKit: cache mounts are per-builder, prune via `docker buildx prune`

> **Don't share cache across users.** The cache contains hardlinks. Two users with permission mismatches will see ghost-files. *Per-user, per-host, fast disk.*

---

## Slide 18 — Migration Playbook — Poetry → uv

### Day 1 — Shadow run

1. Install uv on every dev machine and CI runner
2. Add a CI job: `uv pip compile pyproject.toml -o /tmp/req.txt` — purely informational, doesn't change behaviour
3. Confirm uv produces a workable resolution against current Poetry constraints

### Day 2 — Convert `pyproject.toml`

```bash
uvx migrate-to-uv     # community tool
```

Or do it by hand: see "Migrating from Poetry" in the introduction deck. Either way:

- Convert `[tool.poetry]` → `[project]`
- Caret `^X.Y` → `>=X.Y,<X+1`
- Move dev groups → `[dependency-groups]`

### Day 3 — Lock & commit

```bash
uv lock
git add pyproject.toml uv.lock
git rm poetry.lock
uv sync --all-groups
uv run pytest -q       # confirm parity
```

Diff `uv pip freeze` before vs after. Resolved versions should match within a patch — investigate any drift.

### Day 4 — Update CI & Docker

- Replace `actions/setup-python` + `poetry install` with `setup-uv@v3` + `uv sync --frozen`
- Switch Dockerfile to `ghcr.io/astral-sh/uv:python3.X-bookworm-slim`
- Drop `poetry` from base images

> **Don't try a big-bang.** Convert one repo, leave it for a week. Get reviewers used to `uv.lock` diffs. Then convert the next.

---

## Slide 19 — Troubleshooting Recipes

### "Resolution failed because…"

- uv prints a human-readable conflict tree — read the chain
- Re-run with `uv lock -v` for the resolver trace
- `uv tree` + `uv tree --invert --package X` shows who pulls X

```bash
uv tree --depth 2
uv tree --invert --package httpx
uv tree --outdated
```

### "Could not find a wheel for X on platform Y"

- Constrain the lock environments: `[tool.uv.environments]`
- Or add the missing wheel index in `[tool.uv.sources]`
- Or relax markers — sometimes a sdist build is fine if the system has the C toolchain

### Verbose / debug output

```bash
uv -v   sync          # info
uv -vv  sync          # debug
uv -vvv sync          # trace
RUST_LOG=uv=trace uv sync
```

### Frequent fixes — paste-ready

```bash
# venv stuck on a stale Python
rm -rf .venv && uv sync

# corrupt cache entry
uv cache clean <package>

# offline rebuild
uv sync --offline --frozen

# force reinstall after C-ext break
uv sync --reinstall-package numpy

# inspect resolved wheel
uv pip show fastapi
```

### When to file an issue

github.com/astral-sh/uv. Astral are extremely responsive — most resolver bugs get fixed within a release cycle.

---

## Slide 20 — Production Cheat Sheet

| Concern | Setting |
|---|---|
| Reproducible install | `uv sync --frozen --no-dev` |
| Bytecode at build | `UV_COMPILE_BYTECODE=1` |
| Hardlink-safe Docker | `UV_LINK_MODE=copy` |
| Pin uv version | `setup-uv@v3 with version: 0.5.4` |
| Pin Python | `.python-version` + `requires-python` |
| Pin git source | `rev = "<sha>"` |
| Lock-drift gate | `uv lock --check` |
| SBOM source | `uv export --generate-hashes` |
| Cache between builds | BuildKit `--mount=type=cache,target=/root/.cache/uv` |
| Private index auth | `UV_INDEX_<NAME>_USERNAME / _PASSWORD` |
| Disable Python downloads | `UV_PYTHON_DOWNLOADS=never` |
| Use system Python only | `UV_PYTHON_PREFERENCE=only-system` |
| Offline build | `UV_OFFLINE=1` |
| Custom cache | `UV_CACHE_DIR=/mnt/cache` |
| Trusted publish | `permissions: id-token: write` |
| OIDC publish | `uv publish --trusted-publishing automatic` |

---

## Slide 21 — Summary & What to Try Next

### What we covered

- Production Dockerfile patterns — single-stage, distroless, multi-stage
- BuildKit cache mounts, dev compose with named-volume venv
- GitHub Actions / GitLab CI / Jenkins / Buildkite recipes
- pre-commit with `pre-commit-uv` + lock-drift hooks
- Realistic monorepo workspace
- Local / git / private-index dependencies
- Reproducibility, supply-chain hashes, SBOM
- PEP 723 scripts at scale
- PyTorch + CUDA wheel selection
- conda / mamba interop, pixi as alternative
- Migration playbook from Poetry — day-by-day
- Troubleshooting recipes & production cheat sheet

### Try this first

1. Rewrite one Dockerfile to use the Astral base + cache mount — measure your build time
2. Add the lock-drift hook to your `.pre-commit-config.yaml`
3. Replace the slowest CI matrix with `setup-uv@v3` + `uv sync --frozen`
4. For the next ML project: declare CUDA wheels in `[tool.uv.sources]` instead of post-install hacks
5. Convert your highest-traffic Poetry project this quarter

### Companion deck

For the conceptual foundations — what uv is, the lockfile model, PEP 723, workspaces — see **"Introduction to uv"**.

### Further reading

- docs.astral.sh/uv/guides
- github.com/astral-sh/uv/discussions
- The uv blog at astral.sh/blog
- pre-commit-uv — github.com/tox-dev/pre-commit-uv
- setup-uv — github.com/astral-sh/setup-uv
