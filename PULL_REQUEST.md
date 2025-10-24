# Fix Docker Multi-Platform Build for linux/amd64 Support

## Problem

The current Docker image publishing workflow fails to create a proper multi-platform manifest, resulting in the following error when pulling the image on linux/amd64 systems:

```bash
docker pull ghcr.io/n7olkachev/db-ui:latest
latest: Pulling from n7olkachev/db-ui
no matching manifest for linux/amd64 in the manifest list entries
```

## Root Cause

The existing GitHub Actions workflow (`.github/workflows/docker-publish.yml`) uses a matrix strategy to build images for different platforms:

```yaml
strategy:
  matrix:
    platform:
      - linux/amd64
      - linux/arm64

steps:
  # ...
  - name: Build and push Docker image
    uses: docker/build-push-action@v5
    with:
      platforms: ${{ matrix.platform }}
```

This approach builds **separate images** for each platform and pushes them individually, rather than creating a **single multi-platform image** with a combined manifest. As a result, Docker cannot find the appropriate platform-specific image when pulling.

## Solution

The fix involves:

1. **Removing the matrix strategy** - Build all platforms in a single job
2. **Adding QEMU setup** - Enable cross-platform emulation for building ARM images on AMD64 runners
3. **Combining platforms** - Specify both platforms in a single build command

### Changes Made

**Before:**
```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
    steps:
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          platforms: ${{ matrix.platform }}
```

**After:**
```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          platforms: linux/amd64,linux/arm64
```

## Benefits

- ✅ Creates a proper multi-platform Docker manifest
- ✅ Supports both `linux/amd64` and `linux/arm64` architectures
- ✅ Single image tag works across all supported platforms
- ✅ Faster workflow execution (single job instead of matrix)
- ✅ Better caching efficiency

## Testing

After this change, users will be able to pull the image on both AMD64 and ARM64 systems:

```bash
# On AMD64 systems
docker pull ghcr.io/n7olkachev/db-ui:latest

# On ARM64 systems (e.g., Apple Silicon)
docker pull ghcr.io/n7olkachev/db-ui:latest
```

Docker will automatically select the correct platform-specific image from the manifest.

## Additional Notes

- The workflow now includes `docker/setup-qemu-action@v3` to enable cross-platform builds
- Build time may increase slightly as both platforms are built sequentially in a single job, but this is offset by better caching
- The change maintains backward compatibility with existing tags and versioning

## Files Changed

- `.github/workflows/docker-publish.yml` - Updated Docker build workflow

## References

- [Docker Buildx Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)
- [GitHub Actions: docker/build-push-action](https://github.com/docker/build-push-action)
- [Docker Manifest Lists](https://docs.docker.com/registry/spec/manifest-v2-2/)

