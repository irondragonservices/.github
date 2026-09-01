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
      dragonguard-private-key: ${{ secrets.DRAGONGUARD_APP_PRIVATE_KEY }}
```

`version-from` names the upstream image whose tag defines this image's
version. `FROM alpine:3.24.1` publishes `3.24.1`, `3.24`, `3` and `latest`, so
a consumer can pin as tightly or as loosely as they want.

For an image built from a source release rather than from somebody else's
image — `iron-snapraid` compiles from a tarball, `iron-squid` comes out of a
Debian package — use `version-arg` instead and name the Dockerfile `ARG` that
holds it. Still one place, still in the Dockerfile.

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
  --certificate-identity-regexp '^https://github\.com/irondragonservices/\.github/\.github/workflows/image-(release|refresh)\.yml@refs/heads/main$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository irondragonservices/iron-alpine
```

The identity is a workflow *in this repository*, not in the image's own — this
is where the signing happens, so this is what the certificate names. That is
worth stating plainly, because the obvious pattern to reach for,
`^https://github.com/irondragonservices/`, accepts a signature from any
workflow in any repository in the organisation. The image's own repository
comes from the `--certificate-github-workflow-repository` claim instead.

Both `image-release` and `image-refresh` sign, so both have to be matched: the
nightly rebuild republishes when the package set has genuinely changed.

SBOM and provenance are attached as attestations:

```sh
docker buildx imagetools inspect ghcr.io/irondragonservices/iron-alpine:3 \
  --format '{{ json .SBOM }}'
```

## DragonGuard

The scan step prefers [DragonGuard](https://github.com/DragonSecurity/dragonguard),
which normalises Trivy and the other engines into one finding schema, scores
each finding against the asset's context, and gates on a baseline rather than
on a raw severity count.

DragonGuard is private, and it stays private. `actions/dragon-scan` reads it by
minting a short-lived installation token for the **dragonsecurity-ci** GitHub
App (app `4777161`), which is installed on `DragonSecurity` with `contents:read`
and on this organisation. The token is scoped to the one repository, expires
within the hour, and belongs to no one — so unlike a personal access token it
does not need rotating and does not stop working when somebody leaves.

To turn it on, set one organisation secret:

| Secret | Value |
|---|---|
| `DRAGONGUARD_APP_PRIVATE_KEY` | The PEM private key for the `dragonsecurity-ci` app |

Without it the action logs a notice and falls back to raw Trivy, rather than
failing a build over a missing secret. Findings are then unscored and ungated
but still block on `CRITICAL`/`HIGH` and still upload SARIF.

The app id is an input with a default, so a different app can be used by
passing `app-id` — the installation has to exist on `DragonSecurity` with
`contents:read` for the token request to succeed.

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

## The one image that does not use any of this

`iron-nessus` gets hadolint and nothing else. Nessus is licensed per
activation, so a built image contains an activated installation tied to
somebody's licence, and every workflow here ends in a push to GHCR. Its README
says so at the top.
