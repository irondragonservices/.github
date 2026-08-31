# irondragonservices/.github

Shared CI for the hardened image fleet. Every image repository in this
organisation is three caller workflows and a Dockerfile; everything else lives
here, so a fix reaches all of them on the next run rather than as ten
identical commits.

Forked from [ironpeakservices](https://github.com/ironpeakservices), whose
images are the origin of the hardening steps. What is different is the
automation around keeping them current.

## What is here

| Path | What it is |
|---|---|
| `.github/workflows/image-pr.yml` | Pull request gate: hadolint, a build on every architecture, and a scan that blocks. |
| `.github/workflows/image-release.yml` | Build, push to GHCR with SBOM and provenance, sign with cosign, re-scan the published artefact, cut a release. |
| `.github/workflows/image-refresh.yml` | Scheduled. Rebuilds cache-free and republishes only when the package set actually moved; re-scans what is published and tracks findings in one issue. |
| `actions/dragon-scan/` | The scan step. Runs [DragonGuard](https://github.com/DragonSecurity/dragonguard) when it can, Trivy when it cannot. |
| `templates/` | The caller workflows to drop into a new image repository. |

## How an image repository uses it

```yaml
# .github/workflows/release.yml
jobs:
  ci:
    uses: irondragonservices/.github/.github/workflows/image-release.yml@main
    with:
      version-from: alpine
    secrets:
      dragonguard-token: ${{ secrets.DRAGONGUARD_TOKEN }}
```

`version-from` names the upstream image whose tag defines this image's
version. `FROM alpine:3.24.1` publishes `3.24.1`, `3.24`, `3` and `latest`, so
a consumer can pin as tightly or as loosely as they want.

## How an image stays current

Four mechanisms, each covering what the others cannot see.

**Renovate** pins every base image to a digest and every action to a SHA
(`DragonSecurity/renovate-presets`). A new upstream digest arrives as a pull
request; digest and pin updates auto-merge as soon as the gate is green. This
is what catches the base image moving.

**The pull request gate** is what makes that auto-merge safe. Renovate is
configured with `ignoreTests: false`, so it waits for the checks — which means
the checks have to exist and have to be required by a ruleset, or there is
nothing to wait for and a broken bump merges itself.

**The scheduled refresh** covers packages the Dockerfile installs itself.
`apk add ca-certificates` picks up a new certificate bundle without the base
image tag or digest moving, so nothing triggers a rebuild and the published
image drifts. The refresh rebuilds with the layer cache disabled, compares the
package set against what is published, and republishes only on a real
difference — digests are not comparable, since a rebuild changes them whether
or not anything inside changed.

**The re-scan** covers time. A CVE disclosed today against an image built last
month changes nothing in the repository, so nothing would otherwise look at it
again. Every refresh run re-scans the published image and keeps a single
issue up to date rather than opening one per run.

## Verifying what you pull

Images are signed keylessly, so there is no key to leak or rotate — the
signature is bound to the workflow identity that produced it.

```sh
cosign verify ghcr.io/irondragonservices/iron-alpine:3 \
  --certificate-identity-regexp '^https://github.com/irondragonservices/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

SBOM and provenance are attached as attestations:

```sh
docker buildx imagetools inspect ghcr.io/irondragonservices/iron-alpine:3 \
  --format '{{ json .SBOM }}'
```

## DragonGuard

The scan step prefers [DragonGuard](https://github.com/DragonSecurity/dragonguard),
which normalises Trivy and the other engines into one finding schema, scores
each finding against the asset's context, and gates on a baseline rather than
on a raw severity count. It is private and pre-release, so `actions/dragon-scan`
degrades to raw Trivy when `DRAGONGUARD_TOKEN` is not set rather than failing a
build over a missing secret. When DragonGuard publishes a binary, that one file
changes and the fleet picks it up.

Set the org secret `DRAGONGUARD_TOKEN` to a token with read access to
`DragonSecurity/dragonguard` to turn it on.

## Adding an image repository

1. Declare it in
   [`github-user-management`](https://github.com/internal-DragonSecurity/github-user-management)
   under `dragonsecurity/irondragonservices/teams/admins.yaml`. That is what
   creates the repository — `create_repo: true` in the org's `app.yaml`.
2. Add a ruleset entry in `dragonsecurity/irondragonservices/repos.yaml`
   requiring `ci / lint` and `ci / build`, or Renovate's auto-merge has
   nothing to wait for.
3. Copy `templates/` into the repository's `.github/workflows/`, replacing
   `UPSTREAM` with the base image name and giving the refresh cron a slot
   nothing else is using.
