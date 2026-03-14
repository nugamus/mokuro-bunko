# syntax=docker/dockerfile:1
#
# Mokuro Bunko — TrueNAS
#
# UID 568, config baked in, zero setup needed.
# ghcr.io/nugamus/mokuro-bunko:latest

# ── Build ────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libffi-dev && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml README.md ./
COPY src/ src/

RUN pip install --no-cache-dir build && python -m build --wheel

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir dist/*.whl


# ── Runtime ──────────────────────────────────────────────
FROM python:3.12-slim

LABEL org.opencontainers.image.title="Mokuro Bunko (TrueNAS)"
LABEL org.opencontainers.image.source="https://github.com/Gnathonic/mokuro-bunko"
LABEL org.opencontainers.image.licenses="MPL-2.0"

RUN apt-get update && apt-get install -y --no-install-recommends \
    tini p7zip-full && rm -rf /var/lib/apt/lists/*

# TrueNAS apps user
RUN groupadd -g 568 apps && \
    useradd -u 568 -g 568 -m -s /bin/bash apps

COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:${PATH}"

# Bake config into the image — no config volume needed
# Override by mounting your own file to /config/config.yaml
COPY deploy/truenas/config.yaml /config/config.yaml

RUN mkdir -p /data && chown -R 568:568 /data /config

ENV PYTHONUNBUFFERED="1" \
    MOKURO_HOST="0.0.0.0" \
    MOKURO_PORT="8080" \
    MOKURO_STORAGE="/data" \
    MOKURO_CONFIG="/config/config.yaml" \
    MOKURO_OCR_BACKEND="skip"

USER 568:568

EXPOSE 8080
VOLUME ["/data"]

HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/', timeout=5)" || exit 1

ENTRYPOINT ["/usr/bin/tini", "--", "mokuro-bunko"]
CMD ["serve"]
