FROM pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        bubblewrap \
        ca-certificates \
        curl \
        nodejs \
        npm \
        git \
        openssh-server \
    && curl -fsSL https://chatgpt.com/codex/install.sh | sh \
    && mkdir -p /opt \
    && cp -a /root/.codex/packages/standalone /opt/codex-standalone \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="/root/.local/bin:${PATH}"

COPY requirements.txt /tmp/requirements.txt

RUN python -m pip install \
    --break-system-packages \
    --no-cache-dir \
    -r /tmp/requirements.txt
