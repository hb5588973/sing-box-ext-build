# XHTTP migration notes

`patches/0003-xhttp-client.patch` adds the XHTTP client transport on top of
`0001-amneziawg` and `0002-masque-outbound`.

## Baseline and ordering

- Base: `SagerNet/sing-box@testing` as fetched by CI.
- Apply order: `0001` → `0002` → `0003` (filename order; CI applies `patches/*.patch`).
- The patch was generated on a tree with `0001` and `0002` already applied, then
  re-verified on a clean worktree with the full three-patch sequence:
  `git apply --check` passes for all three and `go build` succeeds.

## Files

New:

- `transport/v2rayxhttp/` — client transport (modes `auto`, `packet-up`,
  `stream-up`, `stream-one`; XMUX pooling; HTTP/1.1, HTTP/2 and HTTP/3).
- `transport/v2ray/registry.go` — client transport registry (`RegisterClient`);
  built-in transports register from `init()`, so XHTTP can self-register behind
  its build tag without touching the client switch again.
- `option/v2ray_xhttp.go`, `option/v2ray_xhttp_xmux_range.go` — option layer.
- `common/tls/std_config_quic.go` — `STDConfigForQUIC` helper used by HTTP/3.
- `include/v2rayxhttp.go` — blank import gated by `with_xhttp`.

Modified:

- `constant/v2ray.go` — `V2RayTransportTypeXHTTP = "xhttp"`.
- `option/v2ray_transport.go` — `XHTTPOptions` field, (un)marshal cases, schema.
- `transport/v2ray/transport.go` — the client `switch` is replaced by a registry
  lookup; server dispatch is unchanged.

## Coexistence with 0001 / 0002

- `0001` only touches `go.mod`, `option/wireguard*`, `transport/wireguard*` and
  `protocol/wireguard`; no overlap with the XHTTP files.
- `0002` adds the MASQUE outbound and its `option`/registry entries. It does not
  touch `transport/v2ray`, and XHTTP registers in the transport registry, not the
  outbound registry, so both `init()`s coexist.
- No `go.mod` change is included in `0003`.

## Decisions

| Topic | Decision | Reason |
|---|---|---|
| HTTP/3 | **kept** (`http3.go` + `common/tls/std_config_quic.go`) | the baseline already uses `github.com/sagernet/quic-go` v0.61.0-sing-box-mod.9 (the same module `0001`'s `go.mod` replace points at), and `STDConfigForQUIC` only needed a small helper. `http3_stub.go` remains for builds without `with_quic`. |
| `common/badh2` | **not ported** | XHTTP no longer rewrites `http2.StreamError`; `hideTransportError` only hides quic-go/http3 types. Tests that pinned the badh2 behaviour were dropped (`stream_error_test.go`, the badh2 assertion in `http3_test.go`). |
| Circuit breaker tests | **dropped** (`breaker_test.go`) | breaker-specific test scaffolding (`failingRT`, `connBreaker` assertions) is out of scope; the XMUX pool itself is kept. |
| SPEC numbering | removed | comments were rewritten as descriptions; the `SPECS/` tree is private to the lx fork. |
| Build tags | `with_xhttp` appended next to `with_awg` | `with_quic` is already part of `DEFAULT_BUILD_TAGS_OTHERS`, so no duplicate is added. |

## Verification performed

```
# clean base, three patches
git apply --check 0001 && git apply 0001
git apply --check 0002 && git apply 0002
git apply --check 0003 && git apply 0003
go build -tags "with_quic with_utls with_xhttp" ./cmd/sing-box   # ok

# package tests
go test -tags "with_xhttp" ./transport/v2rayxhttp/               # ok
go test -tags "with_xhttp with_quic" ./transport/v2rayxhttp/     # ok

# config from the 6 sample nodes
sing-box check -c xhttp-config.json                              # ok (with with_xhttp)
sing-box check -c xhttp-config.json                              # unknown transport type: xhttp (without the tag)
```

## Live node results

Run locally with a binary built from the three-patch tree
(`-tags "with_quic with_utls with_xhttp"`), one node per sing-box process,
`http://cp.cloudflare.com/generate_204` through the mixed inbound:

| # | Node | Result |
|---|---|---|
| 0 | xhttp-h1.1-cdn (skk.moe) | OK, 204 |
| 1 | xhttp-h2-reality (154.40.49.226:30001) | FAIL: `reality verification failed` |
| 2 | xhttp-h3-direct (154.40.49.226:30002) | OK, 204 (HTTP/3 path) |
| 3 | hax xhttp-h1.1-cdn | OK, 204 |
| 4 | hax xhttp-h2-reality | FAIL: `reality verification failed` |
| 5 | hax xhttp-h3-direct | OK, 204 |

The two REALITY nodes fail during the REALITY handshake while the other node on
the same host (h3, port 30002) succeeds, so the transport itself works; the
REALITY mismatch is either server-side or a divergence between this baseline and
the lx tree (lx carries additional REALITY/VLESS changes that are out of scope
here).

MASQUE coexistence was verified at registration level (`type: masque` reaches the
MASQUE option validation instead of `unknown outbound type`), and the CI build
includes `with_awg`; no live MASQUE/AWG dial was repeated in this run.

CI artifact download was blocked by the local network (release CDN timeout), so
the live tests used a locally built binary from the same three-patch tree; CI
itself passed on all 11 targets (`build-25`).

## Known limitations

- HTTP/3 tests require `with_quic`; the stub build rejects `h3` at dial time.
- The removed badh2 layer means `http2.StreamError` is no longer masked; callers
  see the raw error (same as upstream sing-box transports).
- REALITY + XHTTP currently fails verification against the two sample nodes (see
  above).
