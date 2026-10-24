# images/hermes-agent/Dockerfile
#
# ghcr.io/ubc/hermes-agent is the operator's published agent image. It
# IS the upstream NousResearch/hermes-agent image, an s6-overlay runtime that
# bundles the gateway, the dashboard, the OpenAI-compatible API server, a
# Playwright/Chromium browser, node, and every Python dependency — with operator
# metadata layered on top.
#
# Why FROM-upstream instead of building the venv ourselves: the agent is designed
# to run under s6 supervision (/init as PID 1, per-profile gateways, profile
# reconcile on boot, browser at /opt/hermes/.playwright). The previous hand-rolled
# `uv sync` image reproduced none of that, so no HermesInstance could reach Ready
# (see #89). The operator now orchestrates the upstream image declaratively: it
# runs `hermes gateway run` with the API server enabled and probes /health on the
# gateway port (internal/resources/statefulset.go).
#
# Pinned by digest for reproducibility + supply chain. The daily
# agent-image-update workflow bumps both HERMES_VERSION and the FROM digest
# when upstream cuts a new release. To bump by hand, resolve a new digest for
# the desired upstream release and update both lines:
#   docker buildx imagetools inspect nousresearch/hermes-agent:<tag>
#
# Build arg:
#   HERMES_VERSION : the upstream release this digest corresponds to. Surfaced as
#                    the hermes.agent/version label the autoupdate controller reads.
ARG HERMES_VERSION=v2026.9.24-ca2

# Multi-arch (linux/amd64 + linux/arm64) manifest-list digest of the upstream
# release recorded in HERMES_VERSION above. buildx selects the per-platform
# image from the list at build time.
FROM nousresearch/hermes-agent@sha256:fca358f12efd65bfaaca05884166f15c0e2788375ca30d77061ac1ebc96452b7

ARG HERMES_VERSION

# CA trust fix for CVE-2026-111943 (eMudhra intermediate CA keys compromised).
# Upstream images ship Debian ca-certificates 20250419 and certifi 2026.05.20,
# which still trust emSign ECC Root CA - G3 and AAA Certificate Services, so the
# agent's Slack, LLM, GitHub and Google calls accept forged certificates. The
# Debian update (deb13u1) and certifi 2026.10.19 drop them. `>=` keeps a newer
# upstream certifi; the final checks fail the build if either fix is missing.
RUN apt-get update \
 && apt-get install -y --no-install-recommends --only-upgrade ca-certificates \
 && rm -rf /var/lib/apt/lists/* \
 && uv pip install --python /opt/hermes/.venv/bin/python --no-cache 'certifi>=2026.10.19' \
 && test ! -e /usr/share/ca-certificates/mozilla/emSign_ECC_Root_CA_-_G3.crt \
 && /opt/hermes/.venv/bin/python -c 'import certifi, sys; sys.exit(certifi.__version__ < "2026.10.19")'

# BC stays on UTC-7 from 2026-11-01 (no fall back). Upstream ships Debian tzdata 2026a
# and pytz 2025.2, which both still have America/Vancouver falling back to UTC-8, so the
# agent's "now", its America/Vancouver cron jobs and the on-call calendar sync would be
# an hour off. trixie-updates' tzdata (2026b+) and pytz 2026.2+ have the change; the
# check fails the build if zoneinfo or pytz still gives UTC-8 in December 2026.
RUN apt-get update \
 && apt-get install -y --no-install-recommends --only-upgrade tzdata \
 && rm -rf /var/lib/apt/lists/* \
 && uv pip install --python /opt/hermes/.venv/bin/python --no-cache 'pytz>=2026.2' \
 && /opt/hermes/.venv/bin/python -c 'import datetime as d, sys, pytz; from zoneinfo import ZoneInfo; t = d.datetime(2026, 12, 1, 12); z = t.replace(tzinfo=ZoneInfo("America/Vancouver")).utcoffset(); p = pytz.timezone("America/Vancouver").localize(t).utcoffset(); sys.exit(0 if z == p == d.timedelta(hours=-7) else 1)'

# grpc ignores certifi and reads its own bundled roots.pem; use the system store.
ENV GRPC_DEFAULT_SSL_ROOTS_FILE_PATH=/etc/ssl/certs/ca-certificates.crt

# Operator metadata. hermes.agent/version is what the autoupdate controller
# (Plan 5) compares against the registry tag. The image's upstream ENTRYPOINT
# (/init → main-wrapper) and env (HERMES_HOME=/opt/data, ...) are inherited.
LABEL org.opencontainers.image.title="hermes-agent" \
      org.opencontainers.image.source="https://github.com/ubc/hermes-operator" \
      org.opencontainers.image.documentation="https://github.com/ubc/hermes-operator/blob/main/images/hermes-agent/README.md" \
      org.opencontainers.image.description="Operator-published NousResearch/hermes-agent (s6 runtime: gateway + API server + dashboard + browser) for hermes-operator" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.vendor="ubc" \
      org.opencontainers.image.base.name="docker.io/nousresearch/hermes-agent" \
      hermes.agent/version="${HERMES_VERSION}"
