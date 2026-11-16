# Verifier environment (separate from the agent's): the official MBABench 2.0
# judge, unmodified, on the LibreOffice build its deterministic checks were
# verified against (judge/detchecks/core/recalc.py: LO_VERIFIED_VERSION 25.8.7).
# The agent's /workspace/solution.xlsx arrives at runtime via task.toml
# `artifacts`; this build context (tests/) holds the task's judge data root.
FROM python:3.12-slim-bookworm

ARG LIBREOFFICE_VERSION=25.8.7.3
ARG MBABENCH_REPO={upstream_repo}
ARG MBABENCH_COMMIT={upstream_commit}

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl \
        fontconfig fonts-dejavu-core fonts-liberation \
        libcairo2 libcups2 libdbus-1-3 libglib2.0-0 libnss3 libsm6 libice6 \
        libx11-6 libx11-xcb1 libxext6 libxinerama1 libxrandr2 libxrender1 \
    && rm -rf /var/lib/apt/lists/*

# LibreOffice from The Document Foundation's archive (Debian ships 7.4, which
# cannot evaluate XLOOKUP / LET / FILTER and would misgrade such workbooks).
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
        amd64) lo_dir=x86_64; lo_arch=x86-64 ;; \
        arm64) lo_dir=aarch64; lo_arch=aarch64 ;; \
        *) echo "unsupported architecture" >&2; exit 1 ;; \
    esac; \
    tarball="LibreOffice_${LIBREOFFICE_VERSION}_Linux_${lo_arch}_deb.tar.gz"; \
    curl -fsSL -o "/tmp/${tarball}" \
        "https://downloadarchive.documentfoundation.org/libreoffice/old/${LIBREOFFICE_VERSION}/deb/${lo_dir}/${tarball}"; \
    tar -xzf "/tmp/${tarball}" -C /tmp; \
    apt-get update; \
    apt-get install -y --no-install-recommends /tmp/LibreOffice_${LIBREOFFICE_VERSION}*/DEBS/*.deb; \
    rm -rf /tmp/LibreOffice_* /var/lib/apt/lists/*; \
    ln -s "$(ls -d /opt/libreoffice*/program/soffice)" /usr/local/bin/soffice; \
    soffice --version

# The judge at the pinned upstream commit, in the environment upstream grades
# from: the root workspace's locked dependency set (setup.sh runs `uv sync` at
# the repo root). Not just the excel-judge member: the judge imports httpx,
# which only other workspace members bring in.
COPY --from=ghcr.io/astral-sh/uv:0.11.14 /uv /usr/local/bin/uv
RUN set -eux; \
    mkdir -p /opt/mbabench; \
    curl -fsSL "${MBABENCH_REPO}/archive/${MBABENCH_COMMIT}.tar.gz" \
        | tar -xz -C /opt/mbabench --strip-components=1; \
    uv export --project /opt/mbabench --frozen \
        --no-dev --no-emit-workspace -o /tmp/judge-requirements.txt; \
    uv pip install --system --no-cache -r /tmp/judge-requirements.txt; \
    rm /tmp/judge-requirements.txt

ENV LIBREOFFICE_PATH=/usr/local/bin/soffice \
    MBABENCH_ROOT=/opt/mbabench \
    PYTHONUNBUFFERED=1

COPY . /tests/
RUN chmod +x /tests/test.sh && rm -f /tests/Dockerfile
