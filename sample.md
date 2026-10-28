Here are the standard sing-box outbound formats for the 12 protocols 
---

## 1. VLESS

```json
{
  "type": "vless",
  "tag": "vless-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "uuid": "UUID",
  "flow": "",
  "packet_encoding": "xudp",
  "tls": {
    "enabled": true,
    "server_name": "EXAMPLE.COM",
    "utls": { "enabled": true, "fingerprint": "chrome" }
  },
  "transport": { "type": "ws", "path": "/ws", "headers": { "Host": "EXAMPLE.COM" } }
}
```
`flow` supports only `xtls-rprx-vision` or empty. For REALITY, add a `reality` block under `tls`.

---

## 2. VMess

```json
{
  "type": "vmess",
  "tag": "vmess-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "uuid": "UUID",
  "security": "auto",
  "alter_id": 0,
  "packet_encoding": "xudp",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM" },
  "transport": { "type": "ws", "path": "/ws", "headers": { "Host": "EXAMPLE.COM" } }
}
```
`security` accepts `auto`, `none`, `zero`, `aes-128-gcm`, `chacha20-poly1305`.

---

## 3. Shadowsocks

```json
{
  "type": "shadowsocks",
  "tag": "ss-out",
  "server": "EXAMPLE.COM",
  "server_port": 8388,
  "method": "2022-blake3-aes-128-gcm",
  "password": "PASSWORD"
}
```
Shadowsocks normally does not layer TLS; add `transport` as needed.

---

## 4. Trojan

```json
{
  "type": "trojan",
  "tag": "trojan-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "password": "PASSWORD",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM" },
  "transport": { "type": "grpc", "service_name": "GRPC_SERVICE" }
}
```
Trojan requires TLS by default; the `tls` block is usually mandatory.

---

## 5. Hysteria

```json
{
  "type": "hysteria",
  "tag": "hy1-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "up_mbps": 100,
  "down_mbps": 100,
  "auth_str": "AUTH_STR",
  "obfs": "OBFS_PASSWORD",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM", "alpn": ["h3"] }
}
```
The string forms `up`/`down` (e.g. `"100 Mbps"`) are also accepted; `up_mbps`/`down_mbps` are their integer-Mbps equivalents.

---

## 6. Hysteria2

```json
{
  "type": "hysteria2",
  "tag": "hy2-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "password": "PASSWORD",
  "up_mbps": 100,
  "down_mbps": 100,
  "obfs": { "type": "salamander", "password": "OBFS_PASSWORD" },
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM", "alpn": ["h3"] }
}
```
`obfs.type` supports only `salamander`. `server_ports` can replace `server_port` for port hopping.

---

## 7. TUIC

```json
{
  "type": "tuic",
  "tag": "tuic-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "uuid": "UUID",
  "password": "PASSWORD",
  "congestion_control": "bbr",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM", "alpn": ["h3"] }
}
```
`congestion_control` accepts `cubic`, `new_reno`, `bbr`.

---

## 8. AnyTLS

```json
{
  "type": "anytls",
  "tag": "anytls-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "password": "PASSWORD",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM" }
}
```
Optional fields: `idle_session_check_interval`, `idle_session_timeout`, `min_idle_session`.

---

## 9. WireGuard (including AWG)

Standard WireGuard outbound (single-peer form):

```json
{
  "endpoints": [
    {
      "type": "wireguard",
      "tag": "warp-awg-1",
      "address": ["172.16.0.2/32"],
      "private_key": "PRIVATE_KEY",
      "mtu": 1280,
      "system": false,
      "peers": [
        {
          "address": "PEER_HOST",
          "port": 2408,
          "public_key": "PEER_PUBLIC_KEY",
          "allowed_ips": ["0.0.0.0/0"],
          "persistent_keepalive_interval": 25
        }
      ],
      "jc": 6, "jmin": 10, "jmax": 50,
      "s1": 0, "s2": 0, "s3": 0, "s4": 0,
      "h1": 1, "h2": 2, "h3": 3, "h4": 4,
      "i1": "<r 2><b 0x...>"
    }
  ]
}
```

AWG fields `jc`/`jmin`/`jmax`/`s1`/`s2`/`s3`/`s4`/`h1`-`h4`/`i1`-`i5` are added directly at the top level. Note: sing-box has marked the WireGuard outbound as deprecated (since 1.11.0) and recommends migrating to `endpoints`. `local_address` is required — not `ip`.

---

## 10. MASQUE

sing-box officially has **no** `masque` outbound type. The `masque` in your repository is a custom type introduced by `0002-masque-outbound.patch`; the format follows that patch's option definitions. Xray's MASQUE config structure can serve as a reference, but field names differ (Xray uses `address`/`port`/`remoteDNS`, etc.).

Based on your converter's mapping logic, an approximate format is:

```json
{
  "type": "masque",
  "tag": "masque-out",
  "server": "EXAMPLE.COM",
  "server_port": 2408,
  "private_key": "PRIVATE_KEY",
  "public_key": "PUBLIC_KEY",
  "ip": "172.16.0.2/32",
  "mtu": 1280,
  "profile": "cloudflare",
  "tls": { "enabled": true, "server_name": "consumer-masque.cloudflareclient.com" }
}
```
**The actual fields follow the `0002` patch source**; the above only reflects the converter's output shape.

---

## 11. Naive

```json
{
  "type": "naive",
  "tag": "naive-out",
  "server": "EXAMPLE.COM",
  "server_port": 443,
  "username": "USERNAME",
  "password": "PASSWORD",
  "tls": { "enabled": true, "server_name": "EXAMPLE.COM" }
}
```
`extra_headers` is optional and takes an array of strings.

---

## 12. SOCKS

```json
{
  "type": "socks",
  "tag": "socks-out",
  "server": "EXAMPLE.COM",
  "server_port": 1080,
  "version": "5",
  "username": "USERNAME",
  "password": "PASSWORD"
}
```
`version` accepts `"4"`, `"4a"`, `"5"`; defaults to 5.

---

## Notes

- **`tag`**: common to all protocols; optional but recommended.
- **`tls` block**: required or optional for VLESS/VMess/Trojan/Hysteria/Hysteria2/TUIC/AnyTLS/Naive; Shadowsocks usually does not need it; WireGuard and MASQUE do not use the standard `tls` block (MASQUE's TLS lives in the custom implementation).
- **`transport` block**: VLESS/VMess/Trojan can layer `ws`/`grpc`/`http`/`httpupgrade`/`xhttp`, etc.; Hysteria series, TUIC, and AnyTLS do not use sing-box's standard `transport` mechanism.
- **`packet_encoding`**: applies only to VLESS/VMess; accepts `""`, `"packetaddr"`, `"xudp"`.
