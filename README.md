# 🧪 uv in Practice

A companion to **Introduction to uv** — production recipes for shipping uv-managed Python services and ML workloads. Distroless Docker images, GitHub Actions caching, monorepo workspaces, PyTorch/CUDA wheel selection, and a Poetry-to-uv migration playbook.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/uv_in_Practice/)

## 📄 [Markdown Version](presentation.md)

## 🐍 [Companion deck — Introduction to uv](https://brendanjameslynskey.github.io/Introduction_to_uv/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | The build → lock → test → image → ship pipeline |
| 02 | Topics | Map of containers, CI/CD, workflows, ML & migration |
| 03 | Production Dockerfile | Single-stage baseline with BuildKit cache mount |
| 04 | Multi-stage distroless | Builder → ~95 MB final image, image-size table |
| 05 | Dev workflow | Compose with named-volume `.venv` and watch mode |
| 06 | GitHub Actions | Matrix across OS+Python, lock-drift gate, concurrency |
| 07 | GitLab/Jenkins/Buildkite | Equivalent recipes outside GitHub |
| 08 | pre-commit | `pre-commit-uv`, lock-drift hook, ruff/mypy bundle |
| 09 | Monorepo | Realistic workspace — apps, libs, change-detection CI |
| 10 | Local / editable / git | `[tool.uv.sources]` overrides without touching `[project]` |
| 11 | Private indexes | Priority levels, env-var auth, AWS CodeArtifact |
| 12 | Reproducibility | Hashes, SBOM, pinning every layer of the stack |
| 13 | PEP 723 at scale | Self-contained scripts as a `scripts/` directory |
| 14 | Jupyter / IPython | Project-scoped kernels, one-shot `uvx jupyter`, jupytext |
| 15 | PyTorch + CUDA | Marker-conditional wheel sources, GPU Docker recipe |
| 16 | conda / mamba | When to keep conda, how uv fits inside, pixi |
| 17 | Benchmarks | 110-package service: cold/warm/add timings vs pip & poetry |
| 18 | Cache management | Inspect, prune, move off SSD, CI hygiene |
| 19 | Poetry → uv playbook | Day-by-day migration plan |
| 20 | Troubleshooting | Resolver errors, platform misses, paste-ready fixes |
| 21 | Production cheat sheet | Two-column reference of every prod knob |
| 22 | Summary | Top-five next steps and further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

uv documentation — docs.astral.sh/uv · setup-uv — github.com/astral-sh/setup-uv · uv-pre-commit — github.com/astral-sh/uv-pre-commit · pre-commit-uv — github.com/tox-dev/pre-commit-uv · python-build-standalone — github.com/indygreg/python-build-standalone · Astral blog — astral.sh/blog · distroless images — github.com/GoogleContainerTools/distroless

## License

Educational use. Code examples provided as-is.
