# sing-box-ext-build

Builds [SagerNet/sing-box](https://github.com/SagerNet/sing-box) with **AmneziaWG** and
**MASQUE** support, straight from upstream sources plus the patches in `patches/`.

- This repository contains **no sing-box source code**.
- `main` holds only `patches/` and `.github/workflows/`.
- The `wireguard-go` branch holds the AmneziaWG fork of `sagernet/wireguard-go`
  (a build dependency, cloned by CI, not sing-box code).

## What a build does

1. clones `SagerNet/sing-box@testing` (latest);
2. clones the `wireguard-go` branch of this repository into
   `src/submodules/wireguard-go` (the local module `go.mod` replacement);
3. applies every `patches/*.patch` with `git apply --check` first — a conflict stops
   the run with `::error::`;
4. computes the build tags **from upstream**:
   `src/release/DEFAULT_BUILD_TAGS_OTHERS` + `,with_awg`;
5. builds `linux/amd64`, `darwin/arm64`, `windows/amd64` with `CGO_ENABLED=0`;
6. publishes `build-<run_number>` with the three binaries.

## Download

Releases: <https://github.com/hb5588973/sing-box-ext-build/releases>

## Build tags

The tag set is never hard-coded: `release/DEFAULT_BUILD_TAGS_OTHERS` is read from the
cloned upstream tree, so official tag changes are followed automatically. Only
`with_awg` is appended.

MASQUE has **no build tag** — the outbound is compiled unconditionally; the workflow
asserts its sources are present instead of asserting a `with_masque` tag (which does
not exist).

`with_naive_outbound` is intentionally absent: it links `sagernet/cronet-go` and
requires a cgo toolchain per platform, which conflicts with CGO-free cross builds.
Official macOS releases enable it; these builds do not.

## Manual trigger

Actions → **build** → *Run workflow*. It also runs daily at 03:00 UTC and on pushes
that touch `patches/**` or the workflow file.

## When upstream changes break a patch

```bash
# 1. update the working copy (D:\Tools\vpn\sing-box), branch testing
git fetch origin
git rebase origin/testing          # resolve conflicts here, keep changes minimal
go build ./... && go test ./transport/masque/...

# 2. re-export patches (LF endings — do not use PowerShell redirection)
cd D:\Tools\vpn\sing-box
git diff --no-color origin/testing..HEAD -- go.mod option/wireguard.go option/wireguard_awg.go \
  protocol/wireguard/endpoint.go transport/wireguard/endpoint.go transport/wireguard/endpoint_options.go \
  transport/wireguard/device_awg.go transport/wireguard/device_stub_awg.go \
  > D:\Tools\vpn\sb-build\patches\0001-amneziawg.patch
git diff --no-color origin/testing..HEAD -- constant/proxy.go include/registry.go \
  option/masque_outbound.go protocol/masque transport/masque > D:\Tools\vpn\sb-build\patches\0002-masque-outbound.patch

# 3. verify against a clean upstream tree
git worktree add ../sb-apply origin/testing --detach
cd ../sb-apply && git apply --check ../sing-box/patches/*.patch

# 4. commit + push (CI rebuilds automatically)
```

If a conflict needs more than mechanical resolution, stop and inspect: the patches are
deliberately small and readable.

## Layout

```
patches/
  0001-amneziawg.patch        option + transport/wireguard AWG support, go.mod replace
  0002-masque-outbound.patch  MASQUE outbound, connectip package, registration
.github/workflows/build.yml
```
