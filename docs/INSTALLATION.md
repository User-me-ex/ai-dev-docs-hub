# Installation

> Detailed installation guide for AI Dev OS across all supported platforms.

## Overview

AI Dev OS ships as a single self-contained binary. Choose the method that best fits your workflow — pre-built binary, Homebrew, npm, Docker, or building from source. All methods produce the same `aidevos` binary.

## System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **OS** | macOS 13+, Windows 10 (WSL2), Linux kernel 5.4+ | Same |
| **CPU** | 2 cores, x86_64 or aarch64 | 4+ cores |
| **RAM** | 2 GB | 8 GB+ |
| **Disk** | 150 MB | 10 GB+ |
| **GPU** | None required | CUDA/MPS for local inference |
| **Ollama** | Optional (local models) | v0.5+ |

## Installation Methods

| Method | Command | Best for |
|--------|---------|----------|
| **Binary download** | Download from GitHub releases | All platforms, CI/CD |
| **Homebrew** | `brew install aidevos/tap/aidevos` | macOS & Linux |
| **npm** | `npm install -g @aidevos/cli` | Node.js ecosystems |
| **Docker** | `docker pull ghcr.io/aidevos/aidevos` | Containerized environments |
| **Build from source** | `git clone` + `bun run build` | Contributors |

### Binary Download

```bash
curl -LO https://github.com/aidevos/aidevos/releases/latest/download/aidevos_darwin_arm64.tar.gz
tar -xzf aidevos_darwin_arm64.tar.gz
sudo mv aidevos /usr/local/bin/
```

### Homebrew

```bash
brew install aidevos/tap/aidevos
```

Upgrade with: `brew upgrade aidevos/tap/aidevos`

### npm

```bash
npm install -g @aidevos/cli
```

The npm package is a thin wrapper that downloads the correct platform binary on install.

### Docker

```bash
docker pull ghcr.io/aidevos/aidevos:latest
docker run --rm -v ~/.aidevos:/root/.aidevos ghcr.io/aidevos/aidevos run "hello world"
docker run -d -p 8374:8374 -v ~/.aidevos:/root/.aidevos ghcr.io/aidevos/aidevos server start
```

### Build from Source

Prerequisites: Bun 1.2+, Node.js 20+, TypeScript 5.5+

```bash
git clone https://github.com/aidevos/aidevos.git
cd aidevos
bun install
bun run build:binary
```

The compiled binary is written to `dist/aidevos`. See [Local Dev](./LOCAL_DEV.md) for details.

## Platform-Specific Notes

**macOS Apple Silicon**: Binary compiled for `arm64`. If code signing warnings appear, run `xattr -d com.apple.quarantine /usr/local/bin/aidevos`.

**macOS Intel**: Binary compiled for `amd64`. Homebrew installs to `/usr/local/bin` by default.

**Windows (WSL2)**: Install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) with an Ubuntu distribution, then follow the Linux instructions. WSL2 is recommended over native Windows for full model provider compatibility.

**Linux**: Pre-built binaries for `x86_64` and `aarch64`. glibc 2.28+ required (Ubuntu 20.04+, Debian 11+, Fedora 36+). System-wide: `sudo mv aidevos /usr/local/bin/`. Per-user: `mv aidevos ~/.local/bin/`.

## Verifying Installation

```bash
aidevos doctor
```

A passing output:

```
✔ Binary version: 1.0.0 (commit abc1234)
✔ Config: ~/.aidevos/config.toml (valid)
✔ Ollama endpoint: http://localhost:11434 (reachable)
✔ Ollama model: llama3.2:3b (available)
✔ Disk space: 42 GB free
```

For verbose output: `aidevos doctor --verbose`

## Uninstallation

```bash
# Homebrew
brew uninstall aidevos/tap/aidevos

# Manual binary
rm /usr/local/bin/aidevos

# npm
npm uninstall -g @aidevos/cli

# Docker
docker rmi ghcr.io/aidevos/aidevos

# Remove all user data (back up first)
rm -rf ~/.aidevos
```

## Related Documents

- [Getting Started](./GETTING_STARTED.md) — first-run walkthrough
- [Local Dev](./LOCAL_DEV.md) — contributing to AI Dev OS
- [Deployment](./DEPLOYMENT.md) — production/server deployment
- [Configuration](./CONFIGURATION.md) — config reference
- [CLI](./CLI.md) — command reference
- [FAQ](./FAQ.md) — frequently asked questions
