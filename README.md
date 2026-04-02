# docker-gha-cache

**Fast Docker image cache / restore**

Composite GitHub Action for caching Docker image builds using the native [actions/cache](https://github.com/actions/cache) backend.

Split into two actions, build and restore, to allow for fast, skippable, parallelizable build jobs that can run before your main jobs.

---

## Actions

### `bestie/docker-gha-cache/build`

* Builds a Docker image and saves it to the local cache.
* Skips the Docker build entirely on cache hit, making it fast to run every time.

### `bestie/docker-gha-cache/restore`

* Loads a cached image into Docker.
* Fails on cache miss, requires the build action already skipped/succeeded.

---

## Usage

### Recommended Multi-job

```yaml
jobs:
  build-docker-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bestie/docker-gha-cache/build@v1
        with:
          tag: myapp:latest

  compile:
    needs: build-docker-image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bestie/docker-gha-cache/restore@v1
        with:
          tag: myapp:latest
      - run: docker run myapp:latest -v $(pwd):/workdir make
```

### Passing extra build arguments

The build action runs `docker buildx build`, to pass an arbitrary command line option through add it to buildx-args.

```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    buildx-args: |
      --build-arg NODE_ENV=production
      --build-arg APP_VERSION=${{ github.sha }}
```

```yaml
- uses: bestie/docker-gha-cache/build@v1
  with:
    tag: myapp:latest
    buildx-args: --secret id=npmrc,src=$HOME/.npmrc
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

### `build`

| Input | Required | Default | Description |
|---|---|---|---|
| `tag` | ✅ | | Docker image tag |
| `file` | | `Dockerfile` | Path to the Dockerfile |
| `context` | | `.` | Docker build context |
| `platform` | | `linux/amd64` | Target platform |
| `buildx-args` | | | Additional arguments appended to `docker buildx build` |
| `key` | | `<tag>-<platform>-<dockerfile-hash>` | Override the cache key |
| `path` | | `/tmp/<key>.tar` | Override the cache file path |

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
