# Operations Guide

## Health Checks

Three health endpoints are exposed:

| Endpoint | Purpose | Probe mapping |
| --- | --- | --- |
| `/livez` | Always returns 200 if the HTTP server is responsive. | `livenessProbe` |
| `/readyz` | Returns 503 when session count exceeds `HEALTHZ_MAX_SESSIONS`. | `readinessProbe` |
| `/healthz` | **Deprecated** alias of `/readyz`. Will be removed in a future release. | — |

### `/livez`

- Always `200 OK` with `{"status":"ok"}`.
- Use for liveness: the event loop is alive and the server can serve HTTP.

### `/readyz`

- `200 OK` with `{"status":"ok","sessions":<n>}` when healthy.
- `503 Service Unavailable` with `{"status":"unhealthy","reason":"session_limit_exceeded","sessions":<n>}` when active sessions exceed `HEALTHZ_MAX_SESSIONS` (default: 10000).
- Use for readiness: the pod should be removed from the service rotation (no restart) when overloaded.

### `/healthz` (deprecated)

Alias of `/readyz`. Retained for backward compatibility during the 0.7.x cycle.
Will be removed in 0.8.0.

### Kubernetes Probe Configuration

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /readyz
    port: 3000
  initialDelaySeconds: 3
  periodSeconds: 10
```

## Environment Variables

| Variable | Default | Description |
| --- | --- | --- |
| `PORT` | `3000` | HTTP listen port |
| `HOST` | `127.0.0.1` | Bind address. Set to `0.0.0.0` to expose to the network — requires `AUTH_MODE=oauth`, otherwise startup refuses (GHSA-8jr5-6gvj-rfpf). |
| `USE_SSE` | `false` | Enable legacy SSE transport |
| `USE_STREAMABLE_HTTP` | `false` | Enable MCP Streamable HTTP transport |
| `AUTH_MODE` | `pat` (app default) / `oauth` (chart default) | `pat` requires loopback bind. `oauth` accepts any bind and validates `Authorization: Bearer` per request, with sessionId bound to the originating Bearer hash. |
| `GITLAB_API_URL` | `https://gitlab.com/api/v4` | GitLab API base URL |
| `GITLAB_PERSONAL_ACCESS_TOKEN` | — | Required in `pat` mode |
| `GITLAB_READ_ONLY_MODE` | `false` | Restrict to read-only tools |
| `CORS_ALLOW_ORIGINS` | — | Comma-separated allowed origins. Empty = `*` only when `AUTH_MODE=pat` AND bind is loopback. Network-exposed binds require an explicit allowlist or get no `Allow-Origin` header. |
| `HEALTHZ_MAX_SESSIONS` | `10000` | Session count threshold for unhealthy status |

## Auth × bind matrix

The HTTP transports carry no authentication of their own in `AUTH_MODE=pat`. The server's safety properties are a function of two axes — see `SECURITY.md` for the full threat model.

| `AUTH_MODE` | `HOST` | Outcome |
|---|---|---|
| `pat` | loopback (127.0.0.0/8, ::1, localhost) | OK for local dev. Wildcard CORS permitted. |
| `pat` | non-loopback | Refused at startup. CWE-306 / CWE-942. |
| `oauth` | any | OK. `Authorization: Bearer` required on every request. SessionId bound to SHA-256 of the originating Bearer; reuse with a different (or missing) Bearer is rejected with 401. |

## Troubleshooting

### Server returns 401 on every request

- **OAuth mode**: Verify the upstream gateway is forwarding the `Authorization: Bearer <token>` header.
- **PAT mode**: Ensure `GITLAB_PERSONAL_ACCESS_TOKEN` is set and the token has not expired.

### High session count / memory growth

Active sessions are stored in-memory. If session count grows unbounded:

1. Check that clients are properly closing SSE connections or sending `DELETE /mcp`.
2. Consider lowering `HEALTHZ_MAX_SESSIONS` and adding a horizontal pod autoscaler based on the `/readyz` response.

### CORS errors in browser console

In OAuth mode, `Access-Control-Allow-Origin: *` is intentionally **not** sent. Set `CORS_ALLOW_ORIGINS` to the exact origin(s) of your frontend application.

## Release atomicity & recovery

The release pipeline has two independent legs that can succeed or fail independently:

- **`build.yml`** — runs on push/tag and publishes the Docker image to `ghcr.io` when a tag is pushed. Helm chart is packaged and pushed on tag pushes as well.
- **`publish.yml`** — runs only on `release.published` or `workflow_dispatch` and publishes the npm package via OIDC Trusted Publishing.

A successful release means both legs landed: npm has the new version, ghcr has the new image, and (on tag pushes) the Helm chart is in the OCI registry. When one leg fails, the other already-published artifact is **immutable** and must not be rewritten — recovery means re-running the failed leg so the two sides converge.

### Recovery paths

#### npm publish succeeded but Docker push failed

1. Verify npm: `npm view @yoda.digital/gitlab-mcp-server@<version>`
2. Re-run `Build & Publish` against the tag:
   ```bash
   gh run list --workflow=build.yml --branch v<version>
   gh run rerun <run-id> --failed
   ```
3. Verify the image landed:
   ```bash
   docker pull ghcr.io/yoda-digital/mcp-gitlab-server:<version>
   gh api /orgs/yoda-digital/packages/container/mcp-gitlab-server/versions | jq '.[0].metadata.container.tags'
   ```

#### Docker push succeeded but npm publish failed

1. Verify the ghcr image is live (commands above).
2. Inspect why `publish-npm` failed:
   ```bash
   gh run list --workflow=publish.yml --limit 5
   gh run view <publish-run-id> --log-failed
   ```
   Common cause: Trusted Publishing configuration drift on npmjs.com. Check Package settings → Trusted publishers → verify the workflow filename, environment name, and repository all match. See [`reference_npm_publish_404_trap` lessons in memory] for an instance where a misleading `404 PUT` masked an auth-config issue.
3. Re-trigger publish via `workflow_dispatch`:
   ```bash
   gh workflow run publish.yml --ref v<version>
   ```

#### Both legs failed

1. Investigate the root cause before re-triggering (GitHub outage, registry incident, dependency failure). Don't blindly re-run if the underlying environment hasn't recovered.
2. If the tag + release artifacts are still valid, re-trigger both legs as above.
3. If the tag itself is bad (e.g. `version` in `package.json` doesn't match the tag, or the release was published from the wrong SHA): **do NOT delete the tag or release** — they may be cached downstream. Instead, bump the version, create a new tag, and document the skipped version in CHANGELOG.

### Verifying a full release post-recovery

```bash
# npm side
npm view @yoda.digital/gitlab-mcp-server@<version> --json | jq '{version, dist: .dist | {tarball, integrity, signatures}}'
# .dist.signatures present = Sigstore provenance attached via Trusted Publishing

# Container side
docker pull ghcr.io/yoda-digital/mcp-gitlab-server:<version>
docker inspect ghcr.io/yoda-digital/mcp-gitlab-server:<version> --format='{{.Config.Labels}}'
```

Both should report the expected version. Note recovery actions in `CHANGELOG.md` if the convergence took more than the standard ceremony — operators downstream may need that context when bisecting.

## First-publish runbook: ghcr.io package

When `ghcr.io/yoda-digital/mcp-gitlab-server` is first published, GitHub creates the container package in **private** mode by default and does **not** link it to the source repository. Both need to be fixed manually once per package, after the first successful `build.yml` run that pushes an image.

### One-time setup steps

1. After the first successful build, navigate to:
   `https://github.com/orgs/yoda-digital/packages/container/mcp-gitlab-server/settings`
2. Under **Manage Actions access**, add the `mcp-gitlab-server` repository with the **Write** role. This is what lets subsequent CI runs push to this package without an additional OAuth dance.
3. Under **Danger Zone** → **Change visibility**, set to **Public**. Without this, anonymous `docker pull` returns 403.
4. Under **Repository**, link the package to `yoda-digital/mcp-gitlab-server` so the README, license, and source link show up on the package landing page.

### Verification

```bash
# Anonymous pull should succeed
docker pull ghcr.io/yoda-digital/mcp-gitlab-server:latest

# Package metadata should report visibility + source repo
gh api /orgs/yoda-digital/packages/container/mcp-gitlab-server \
  | jq '{visibility, repository: .repository.full_name, html_url: .html_url}'
```

Expected:
```json
{
  "visibility": "public",
  "repository": "yoda-digital/mcp-gitlab-server",
  "html_url": "https://github.com/orgs/yoda-digital/packages/container/package/mcp-gitlab-server"
}
```

### Failure mode: "denied" on subsequent CI pushes

If `build.yml` succeeds initially but later runs start failing with `denied: permission_denied` on the push step, the Actions access from step 2 has dropped. This can happen if:

- The repo was renamed or moved.
- The org's default Actions permissions tightened.
- A maintainer was removed and the package's per-repo grant was tied to their account.

Fix: re-apply step 2.

## Verifying the image (cosign + SBOM + provenance)

Every published image carries a Sigstore keyless signature, a SLSA provenance attestation, and a SPDX SBOM attached to its manifest. Downstream operators — compliance, security review, FedRAMP/ENISA procurement — can verify all three against `ghcr.io`. Verification requires network access to the Sigstore Rekor transparency log; it is not a true offline check.

### Prerequisites

```bash
# Install cosign (https://docs.sigstore.dev/cosign/system_config/installation/)
go install github.com/sigstore/cosign/v2/cmd/cosign@latest
# or via Homebrew:
brew install cosign
```

### Verify the signature

```bash
IMAGE="ghcr.io/yoda-digital/mcp-gitlab-server:latest"

cosign verify \
  --certificate-identity-regexp '^https://github\.com/yoda-digital/mcp-gitlab-server/\.github/workflows/build\.yml@refs/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "$IMAGE"
```

Expected output: a JSON payload listing the signed claim, OIDC identity (`https://github.com/yoda-digital/mcp-gitlab-server/.github/workflows/build.yml@refs/...`), and Rekor transparency-log entry. A non-zero exit means tampering, a Rekor outage, or that you pulled an image not produced by this repo's pipeline.

### Download the SBOM

```bash
cosign download sbom "$IMAGE" > sbom.spdx.json
jq '.packages[] | {name: .name, version: .versionInfo}' sbom.spdx.json | less
```

The SBOM is SPDX 2.3 JSON and lists every OS package, Node module, and base-image layer that contributed to the final image. Use it to answer *"are we affected by CVE-XXXX-YYYY in dependency Z?"* without rebuilding locally.

### Inspect the provenance attestation

```bash
cosign download attestation "$IMAGE" \
  | jq -r '.payload' \
  | base64 -d \
  | jq '.predicate'
```

The provenance follows the SLSA in-toto schema (mode=max) and records the workflow source ref, builder identity, and build inputs.

### Multi-arch manifest

The published manifest contains both `linux/amd64` and `linux/arm64`. Verify with:

```bash
docker manifest inspect "$IMAGE" \
  | jq '.manifests[] | {platform: .platform, digest: .digest}'
```

Apple Silicon, AWS Graviton, and ARM-based Kubernetes nodes will pull the native arm64 layer automatically.

### CI smoke gate

Each `build.yml` run pushes the image, then immediately runs `cosign verify` against the freshly-signed digest. On tag releases (the trust boundary), a failed verify hard-aborts the workflow before downstream consumers can pull the bad artifact. On main and feature branches, verify is retried with backoff and soft-warns (non-blocking) — this absorbs the well-known Rekor inclusion lag (10–30s in pathological cases) without freezing developer iteration.

### Sigstore outage — temporary release path

When Sigstore (Rekor or Fulcio) is impaired and a release must ship anyway:

1. In `.github/workflows/build.yml`, comment out the `Install cosign`, `Sign image with cosign (keyless OIDC)`, and `Verify cosign signature (CI smoke with Rekor lag tolerance)` steps for the duration of the outage. Do **not** delete them.
2. Push the workflow change to `main` BEFORE cutting the tag.
3. In the GitHub Release notes (and `CHANGELOG.md` under `[Unreleased]`), prepend:
   > **Released during Sigstore outage**: this image is **not** cosign-signed. Verify provenance via the SHA256 digest published below and against the GitHub Actions run URL.
4. Publish the image digest from the workflow run summary in the release notes so consumers can pin by digest.
5. Once Sigstore recovers, **re-enable the cosign steps in `build.yml`** (revert step 1), then cut a no-op patch release (`0.x.y+1` with only a `CHANGELOG.md` entry) that ships a signed image of the same code. Announce the resumed signing in that release's notes.

Do not paper over a Sigstore outage with `--insecure-ignore-tlog` — that defeats the supply-chain claim.

### Identity rotation (org rename, workflow file move)

The cosign verify identity regex above is anchored on three things downstream consumers will pin:

- Org name: `yoda-digital`
- Repo name: `mcp-gitlab-server`
- Workflow path: `.github/workflows/build.yml`

Changing any one silently invalidates every consumer's `cosign verify` script. If a move is unavoidable:

1. **Before the move**: open an issue + `CHANGELOG.md` entry under `[Unreleased]` announcing the new identity regex (the post-move URL) one release ahead of time.
2. **After the move**: tag a no-op release immediately so consumers have a known-good signed image under the new identity. Update the example in this section of `OPERATIONS.md`.
3. **For consumers**: maintain a deprecation issue on the old identity for at least 90 days, linking to the new regex. Encourage pinning by `cosign verify --certificate-identity-regexp '<old>|<new>'` during the transition window.
