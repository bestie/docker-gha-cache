# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A set of **composite GitHub Actions** (no Node, no compiled code) that cache Docker image builds via the native [actions/cache](https://github.com/actions/cache) backend, storing images as `.tar` files.

- `action.yml` (repo root) — the **unified** action. A `mode` input (`build` default, or `restore`) selects which flow runs. This is the Marketplace-published entry point. It is **self-contained** — it duplicates the build and restore logic rather than delegating to the subdirectory actions (see the duplication invariant below).
- `build/action.yml` — builds a Docker image and saves the tar to cache. **Skips the build entirely on cache hit** so it's cheap to run on every workflow.
- `restore/action.yml` — loads a cached image tar into the local Docker daemon. Uses `fail-on-cache-miss: true`, so it requires a prior `build` to have populated the cache.

The split exists so `build` jobs can run first, in parallel, and be skipped on cache hit, while downstream jobs `restore` the image. The unified root action exists because the GitHub Marketplace only publishes an action whose `action.yml` is at the repo root.

## Architecture / key invariants

- **Cache key must match across all three actions.** `action.yml`, `build/action.yml`, and `restore/action.yml` each compute the *same* default key inline: `<tag>-<platform>-<hashFiles(file)>`, with default path `/tmp/<key>.tar`. This key/path derivation logic is duplicated in the `init` step of **all three** files — **if you change it in one, change it in all three**, or restore will miss. The duplication is load-bearing: composite actions resolve `uses: ./path` relative to the *consumer's* workspace (open runner bug actions/runner#1348), so the root action cannot delegate to the subdir actions for external consumers — each file must be self-contained.
- `restore` takes `tag`, `platform`, and `file` only because it needs them to recompute the same default key — not to do anything Docker-specific beyond `docker load`.
- `build` uses `actions/cache` with `lookup-only: true` first (the `cache-check` step) to detect a hit, then gates every subsequent step (QEMU, buildx, build, save) on `cache-hit != 'true'`.
- QEMU is set up only when the target `platform` differs from the runner's native arch; buildx is always set up on a cache miss.
- `buildx-args` is a multiline string input appended verbatim to `docker buildx build`. It's normalized line-by-line in a bash `while read` loop (handles empty input and missing trailing newline) — see the "Build args list bug" / "empty build args bug" fixes in git history. Be careful editing this loop.
- `load` (default `true`) adds `--output type=docker` so the image is usable in the same job. The recommended multi-job pattern sets `load: false` for speed.

## Testing

There is **no local test runner** — the actions depend on the live GHA cache service, so tests only run in the Actions environment. To test changes: push to a branch and open a PR.

`.github/workflows/ci.yml` is the end-to-end test suite. It runs on every push and PR, and covers: clean build (cache miss), cross-job restore, repeated-build cache-hit short-circuit, `buildx-args` passthrough, custom `key`/`path` overrides, single-job inline use, and the unified root action in both modes plus its invalid-mode guard. Test fixtures: `tests/Dockerfile` (minimal) and `tests/Dockerfile.args` (exercises `--build-arg` passthrough).

The CI workflow references the actions as local paths (`uses: ./` for the root action, `uses: ./build`, `uses: ./restore`), so it tests the working-tree version.

## Conventions (from CONTRIBUTING.md)

- Keep actions self-contained: no external scripts, no Node dependencies.
- Prefer explicit `if:` conditions over relying on step failure propagation.
- Quote all shell variable expansions.

## Releases

Semantic versioning. After tagging a release, move the floating major tag so `@v1` users get the update:

```bash
git tag -f v1
git push origin v1 --force
```
