# syntax=docker/dockerfile:1
# ROCm 7.14 RDNA4 (gfx1201) image: torch 2.14.1+rocm7.14 wheels are
# self-contained (rocm-sdk-* wheels bundle MIOpen/rocBLAS/hip), so unlike the
# 6.4 image this build needs NO rocm/dev base, NO arch-linux rocBLAS shim, and
# NO repo.radeon MIOpen kdb download (none exist for gfx1201 anyway).
# Only requirement: host kernel driver exposes /dev/kfd + /dev/dri (7.14 host stack).
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data

RUN apt-get update && apt-get install -y --no-install-recommends \
        espeak-ng espeak-ng-data libsndfile1 curl ca-certificates git g++ \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/uv && mv /root/.local/bin/uvx /usr/local/bin/uvx \
    && useradd -m -u 1001 appuser \
    && mkdir -p /app/api/src/models/v1_0 /home/appuser/.config/miopen /home/appuser/.cache/miopen \
    && chown -R appuser:appuser /app /home/appuser

USER appuser
WORKDIR /app

COPY --chown=appuser:appuser pyproject.toml uv.lock ./

ENV UV_COMPILE_BYTECODE=1

RUN --mount=type=cache,target=/home/appuser/.cache/uv,uid=1001,gid=1001 \
    uv venv --python 3.12 && uv sync --frozen --extra rocm7

# Japanese support (fugashi/MeCab UniDic ~526MB); set false to skip
ARG INCLUDE_JAPANESE=true
RUN if [ "$INCLUDE_JAPANESE" = "true" ]; then .venv/bin/python -m unidic download; fi

COPY --chown=appuser:appuser docker/scripts/download_model.py ./download_model.py
COPY --chown=appuser:appuser api/src/models ./api/src/models
ARG DOWNLOAD_MODEL=true
RUN if [ "$DOWNLOAD_MODEL" = "true" ]; then .venv/bin/python download_model.py --output api/src/models/v1_0; fi

COPY --exclude=src/models --chown=appuser:appuser api ./api
COPY --chown=appuser:appuser web ./web
COPY --chown=appuser:appuser VERSION ./VERSION
COPY --chown=appuser:appuser docker/scripts/ ./
COPY --chown=appuser:appuser docker/rocm/warmup_miopen.py ./docker/rocm/warmup_miopen.py
RUN chmod +x ./entrypoint.sh

ENV PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    PATH="/app/.venv/bin:$PATH" \
    UV_LINK_MODE=copy \
    USE_GPU=true \
    DOWNLOAD_MODEL=true \
    DEVICE="gpu" \
    MIOPEN_FIND_MODE=2

CMD ["./entrypoint.sh"]
