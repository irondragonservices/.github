## Iron Dragon Services

Hardened container base images that stay hardened.

Forked from [ironpeakservices](https://github.com/ironpeakservices). The
hardening is theirs; what is added here is the machinery that keeps a
published image current — digest-pinned bases with auto-merged updates, a
cache-free scheduled rebuild that republishes only on a real package change,
and a re-scan that catches vulnerabilities disclosed after the build.

Every published image is multi-architecture, signed with cosign keyless, and
ships an SBOM and provenance attestation. Each repository's README carries the
exact `cosign verify` command for it. `iron-nessus` is the one exception
and is deliberately not published — Nessus is licensed per activation, so a
built image carries somebody's entitlement.

| Image | Base | For |
|---|---|---|
| [iron-alpine](https://github.com/irondragonservices/iron-alpine) | alpine | General purpose, when you need a package manager |
| [iron-debian](https://github.com/irondragonservices/iron-debian) | debian | General purpose, glibc |
| [iron-scratch](https://github.com/irondragonservices/iron-scratch) | scratch | Static binaries — Go, Rust |
| [iron-nginx](https://github.com/irondragonservices/iron-nginx) | distroless | Serving static content |
| [iron-redis](https://github.com/irondragonservices/iron-redis) | distroless | Redis |
| [iron-cockroachdb](https://github.com/irondragonservices/iron-cockroachdb) | scratch | CockroachDB |
| [iron-cloudflared](https://github.com/irondragonservices/iron-cloudflared) | scratch | Cloudflare Tunnel |
| [iron-squid](https://github.com/irondragonservices/iron-squid) | distroless | Squid proxy |
| [iron-snapraid](https://github.com/irondragonservices/iron-snapraid) | scratch | SnapRAID |
| [iron-nessus](https://github.com/irondragonservices/iron-nessus) | debian | Nessus — **not published**, build it yourself |

CI for all of them lives in [`.github`](https://github.com/irondragonservices/.github).
