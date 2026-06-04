# docker-gha-cache v1

**Fast Docker image cache / restore**

Composite GitHub Action for caching Docker image builds using the native [actions/cache](https://github.com/actions/cache) backend.

Available as a single unified action with a `mode` input, or as two focused actions (`build` and `restore`), to allow for fast, skippable, parallelizable build jobs that can run before your main jobs.

---

## Actions

### `bestie/docker-gha-cache` (unified)

A single action that does either job, selected by the `mode` input:

* `mode: build` (default) — builds a Docker image and saves it to the local cache. Skips the Docker build entirely on cache hit, making it fast to run every time.
* `mode: restore` — loads a cached image into Docker. Fails on cache miss, requires a prior `build` to have populated the cache.

This is the recommended entry point and the one published to the GitHub Marketplace.

### `bestie/docker-gha-cache/build`

* Builds a Docker image and saves it to the local cache.
* Skips the Docker build entirely on cache hit, making it fast to run every time.
* Equivalent to the unified action with `mode: build`.

### `bestie/docker-gha-cache/restore`

* Loads a cached image into Docker.
* Fails on cache miss, requires the build action already skipped/succeeded.
* Equivalent to the unified action with `mode: restore`.

---

## Usage

### Unified action

The same action handles both jobs via the `mode` input. `mode` defaults to `build`.

```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Build and cache (skips on cache hit)
        uses: bestie/docker-gha-cache@v1
        with:
          tag: myapp:latest
          load: false

  compile:
    needs: build-docker-image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Load Docker image
        uses: bestie/docker-gha-cache@v1
        with:
          mode: restore
          tag: myapp:latest
      - name: Compile
        run: docker run myapp:latest -v $(pwd):/workdir make
```

All `build` inputs below also apply to the unified action in `build` mode. The split `build`/`restore` actions documented below remain available and unchanged.

### Basic Single Job setup

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Build or load from cache
        uses: bestie/docker-gha-cache/build@v1
        with:
          tag: myapp:latest

      - name: Compile
        run: docker run myapp:latest -v $(pwd):/workdir make
```

### Recommended Multi-Job

Build one or more Docker images in separate, parallelizable jobs.

Dependent jobs use restore to load the image.

```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Build and cache (skips on cache hit)
        uses: bestie/docker-gha-cache/build@v1
        with:
          tag: myapp:latest
          load: false

  lint:
    needs: build-docker-image
    # ...

  compile:
    needs: build-docker-image
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6

      - name: Load Docker image
        uses: bestie/docker-gha-cache/restore@v1
        with:
          tag: myapp:latest

      - name: Compile
        run: docker run myapp:latest -v $(pwd):/workdir make

  test:
    needs: build-docker-image
    # ...
```

### Passing arbitrary arguments to `docker build`

buildx-args is a list of strings that is appended as arguments to the `docker buildx build` command ran by the build action.

This should allow most aspects of the build to be customized.

Add secrets:
```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    buildx-args: |
      --secret id=npmrc,src=$HOME/.npmrc
```

Build args:
```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    buildx-args: |
      --build-arg NODE_ENV=production
      --build-arg APP_VERSION=${{ github.sha }}
```

### Authenticating to a private registry

Pass your authentication command as the `prebuild` input parameter.
It will be run just before docker buildx and skipped on cache hit.

```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    prebuild: |
      echo "${{ secrets.REGISTRY_PASSWORD }}" | \
        docker login ghcr.io -u "${{ github.actor }}" --password-stdin
```

### Cross-platform builds

Set the platform argument to change the target architecture, the following will build an arm64 image on an x86 runner.

```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    platform: linux/arm64
```

QEMU is set up automatically when the target platform differs from the runner's native architecture and skipped if they match.

### Custom cache key

By default the cache key is a function of the image tag, platform, and a hash of the Dockerfile.

For finer control over what triggers a rebuild, define your own cache key.

```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    key: myapp-${{ hashFiles('Dockerfile', 'package-lock.json') }}
```

---

## Inputs

### unified action

The unified action accepts every `build` input below, plus:

| Input  | Required | Default | Description |
|--------|----------|---------|---|
| `mode` |          | `build` | `build` to build and cache the image, or `restore` to load a cached image. In `restore` mode the build-only inputs (`load`, `buildx-args`, `context`, `prebuild`) are ignored. |

### `build`

| Input         | Required  | Default           | Description |
|-------------- |-----------|-------------------|---|
| `tag`         | ✅        |                   | Docker image tag |
| `file`        |           | `Dockerfile`      | Path to the Dockerfile |
| `context`     |           | `.`               | Docker build context |
| `load`        |           | `true`            | If a newly built image is loaded into the local daemon |
| `platform`    |           | `linux/amd64`     | Target platform |
| `buildx-args` |           |                   | Additional arguments appended to `docker buildx build` |
| `prebuild`    |           |                   | Shell command run before the build (e.g. registry login) |
| `key` | | `<tag>-<platform>-<dockerfile-hash>` | Override the cache key |
| `path`        |           | `/tmp/<key>.tar`  | Override the cache file path |

### `restore`

| Input | Required | Default | Description |
|---|---|---|---|
| `tag` | ✅ | | Docker image tag |
| `file` | | `Dockerfile` | Path to the Dockerfile (used to compute default key) |
| `platform` | | `linux/amd64` | Target platform (used to compute default key) |
| `key` | | `<tag>-<platform>-<dockerfile-hash>` | Override the cache key |
| `path` | | `/tmp/<key>.tar` | Override the cache file path |

---

## Outputs

The unified action exposes the same outputs as `build` (`cache-hit`, `key`, `path`) in both modes.

### `build`

| Output | Description |
|---|---|
| `cache-hit` | `'true'` if the image was already cached and the build was skipped |
| `key` | The cache key used |
| `path` | The cache file path used |

### `restore`

| Output | Description |
|---|---|
| `cache-hit` | `'true'` if the image was successfully restored from cache |


---

## Requirements

- The runner must have Docker installed. Standard GitHub-hosted runners (`ubuntu-*`) include this by default.
- Buildx is set up automatically by the build action.
- QEMU is set up automatically when cross-platform builds are needed.

---

## License

MIT
