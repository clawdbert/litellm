# Pinned by digest (resolved from main-stable on 2026-05-08).
# To bump: docker buildx imagetools inspect ghcr.io/berriai/litellm:main-stable
# and replace the digest below.
FROM ghcr.io/berriai/litellm:main-stable@sha256:6c82d338a60e7b273ae46bf1d8db814d2856ae010f96c44eeadde574d3893f76
# The upstream image strips pip from /app/.venv. Bootstrap it via ensurepip
# before installing posthog. We do not keep pip on PATH; this only enables
# the install step.
RUN python -m ensurepip --upgrade \
 && python -m pip install --no-cache-dir posthog
