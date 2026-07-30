FROM pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        bubblewrap \
        ca-certificates \
        curl \
        tmux \
        nodejs \
        npm \
        git \
        openssh-server \
    && curl -fsSL https://chatgpt.com/codex/install.sh \
        | CODEX_HOME=/opt/codex \
          CODEX_INSTALL_DIR=/usr/local/bin \
          CODEX_NON_INTERACTIVE=true \
          sh \
    && rm -rf /var/lib/apt/lists/*

ENV CODEX_HOME="/root/.codex"
COPY requirements.txt /tmp/requirements.txt
RUN python -m pip install \
    --break-system-packages \
    --no-cache-dir \
    -r /tmp/requirements.txt
