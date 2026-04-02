# Contributing

Contributions are welcome. Please open an issue before submitting a pull request for anything beyond small fixes.

## Development

The repo structure is:

```
build/action.yml       # Build and cache action
restore/action.yml     # Restore and load action
tests/
  Dockerfile           # Minimal image used by CI tests
  Dockerfile.args      # Image with ARG, used to test buildx-args passthrough
.github/workflows/
  ci.yml               # End-to-end tests run on every PR and push to main
```

## Testing

Tests run automatically via GitHub Actions on every push and pull request. To test your changes, push to a branch and open a PR — the CI workflow exercises:

- A clean build populating the cache
- A restore in a separate job loading the cached image
- A repeated build confirming the cache-hit short-circuit works
- `buildx-args` passthrough via `--build-arg`
- Custom `key` and `path` overrides

There is no local test runner — the actions depend on the GHA cache service, so testing requires the Actions environment.

## Releases

This project uses [semantic versioning](https://semver.org). When merging to `main`:

- Patch releases (`v1.0.x`) — bug fixes, no input/output changes
- Minor releases (`v1.x.0`) — new optional inputs, backwards compatible
- Major releases (`vX.0.0`) — breaking changes to inputs, outputs, or behaviour

After tagging a release, move the floating major tag (e.g. `v1`) to point to the new commit so users pinned to `@v1` get the update automatically:

```bash
git tag -f v1
git push origin v1 --force
```

## Code style

- Keep the actions self-contained — no external scripts, no Node dependencies.
- Prefer explicit `if:` conditions over relying on step failure propagation.
- Quote all shell variable expansions.
