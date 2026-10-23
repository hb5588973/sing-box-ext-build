# sing-box-ext-build

> Automated builds of [SagerNet/sing-box](https://github.com/SagerNet/sing-box) with **AmneziaWG** and **MASQUE** support.
> Code comes straight from upstream sources plus a set of lightweight patches.

[![Build Status](https://github.com/hb5588973/sing-box-ext-build/actions/workflows/build.yml/badge.svg)](https://github.com/hb5588973/sing-box-ext-build/actions/workflows/build.yml)

## Features

- **No sing-box source code** is stored in this repository.
- Contains only `patches/` and `.github/workflows/`.
- Fetches the latest `testing` branch from upstream every day.
- Applies patches and builds binaries cross-platform with `CGO_ENABLED=0`.
- Automatically tracks official upstream build tags (never hard-coded).
- AmneziaWG support is added via `with_awg` tag.
- MASQUE CONNECT-IP outbound is compiled unconditionally (requires no build tag).

## Download

Latest pre-built binaries: 
👉 **[Download Latest Release](https://github.com/hb5588973/sing-box-ext-build/releases/latest)**

Available platforms: `linux-amd64`, `linux-arm64`, `linux-armv7`, `linux-armv6`, `linux-riscv64`, `linux-mipsle`, `linux-mips`, `linux-mips64le`, `linux-mips64`, `darwin-arm64`, `windows-amd64`.

## How to Fork & Build Your Own

If you want to use this approach for your own fork:

1. **Fork this repository** (e.g., `yourname/sing-box-ext-build`).
2. **Fork the wireguard-go fork** (e.g., `yourname/wireguard-go-awg`) if you have custom AWG changes. *If you don't want to fork it, modify the `WIREGUARD_REPO` env in `.github/workflows/build.yml` to point to the original `hb5588973/wireguard-go-awg`.*
3. The GitHub Actions workflow is pre-configured. You just need to trigger it:
   - Go to the **Actions** tab.
   - Select **build**.
   - Click **Run workflow**.

The workflow will automatically:
1. Clone `SagerNet/sing-box@testing`.
2. Clone `wireguard-go-awg` into `src/submodules/wireguard-go` (local module replacement).
3. Apply `patches/*.patch` (`git apply --check` fails fast on conflicts).
4. Read `src/release/DEFAULT_BUILD_TAGS_OTHERS` from upstream, append `with_awg`.
5. Build with `CGO_ENABLED=0` using upstream's `release/LDFLAGS`.
6. Publish binaries to the rolling `latest` release.

## Build Tags & MASQUE

The tag set is never hard-coded: `release/DEFAULT_BUILD_TAGS_OTHERS` is read from the cloned upstream tree. Only `with_awg` is appended.

MASQUE has **no build tag** — the outbound is compiled unconditionally; the workflow asserts its sources are present instead of asserting a non-existent `with_masque` tag.

`with_naive_outbound` is intentionally absent: it links `sagernet/cronet-go` and requires a cgo toolchain per platform, which conflicts with CGO-free cross builds. Official macOS releases enable it; these builds do not.

## Maintenance: When upstream changes break a patch

If the scheduled CI run fails because upstream changed something that conflicts with our patches:

```bash
# 1. Clone your fork locally and rebase against upstream
git clone https://github.com/YOUR_NAME/sing-box-ext-build.git
cd sing-box-ext-build

# 2. Create a temporary working tree to apply and resolve patches
mkdir work && cd work
git clone --depth 1 --branch testing https://github.com/SagerNet/sing-box.git
cd sing-box
# Apply your patches one by one
git apply ../../patches/0001-amneziawg.patch
# If conflicts occur, fix them, then generate a new patch:
git diff > ../../patches/0001-amneziawg.patch

# 3. Push the updated patches back to your fork
cd ../..
git add patches/
git commit -m "fix: update patches for upstream changes"
git push

## Layout

```
patches/
  0001-amneziawg.patch        option + transport/wireguard AWG support, go.mod replace
  0002-masque-outbound.patch  MASQUE outbound, connectip package, registration
.github/workflows/build.yml
```
