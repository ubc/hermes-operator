# hermes-agent image build context

The operator owns `ghcr.io/ubc/hermes-agent`. It is the **upstream
`nousresearch/hermes-agent` image** (an s6-overlay runtime that bundles the
gateway, dashboard, OpenAI-compatible API server, a Playwright/Chromium browser,
node, and every Python dependency) with operator metadata layered on top.

We intentionally do **not** rebuild the Python environment ourselves. The agent
is designed to run under s6 supervision (`/init` as PID 1, per-profile gateways,
profile reconcile on boot, browser under `/opt/hermes/.playwright`); reproducing
that from a bare `uv sync` is both fragile and incomplete (see #89). The operator
orchestrates the upstream image declaratively: it runs `hermes gateway run` with
the API server enabled and probes `/health` on the gateway port (see
`internal/resources/statefulset.go` and `docs/runtime.md`).

## Layout

| File | Purpose |
|---|---|
| `Dockerfile` | `FROM` the upstream image (pinned by multi-arch digest) + operator OCI/version labels. |
| `README.md` | This file. |

## Bumping the upstream version

This is automated. `.github/workflows/agent-image-update.yaml` runs daily,
compares the latest upstream `NousResearch/hermes-agent` release against the
`HERMES_VERSION` pinned in `Dockerfile`, and on a new release opens (and
auto-merges, when repo settings allow) a PR that bumps both the
`FROM ...@sha256:<digest>` line and the `HERMES_VERSION` build arg. Once the
bump lands on `main`, `.github/workflows/agent-image.yaml` builds, signs
(Cosign keyless), and pushes the new image. The SPDX SBOM ships two ways:
attached in-registry by BuildKit (`sbom: true`) and uploaded as a workflow
run artifact. There is no rekor-logged SBOM attestation; the document is
multiple megabytes and exceeds the public Rekor entry size cap.

To bump by hand instead:

1. Resolve the new multi-arch manifest digest for the desired upstream release:

   ```bash
   docker buildx imagetools inspect nousresearch/hermes-agent:<tag>
   ```

2. Update both the `FROM ...@sha256:<digest>` line and the `HERMES_VERSION` build
   arg (the value surfaced as the `hermes.agent/version` label, which the
   autoupdate controller compares against the registry tag) in `Dockerfile`.

3. Merge to `main`; the push trigger on `agent-image.yaml` publishes the image.
   Tag pushes (`agent/vX.Y.Z`) and manual `workflow_dispatch` also work.
