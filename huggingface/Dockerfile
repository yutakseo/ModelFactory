FROM pytorch/pytorch:2.11.0-cuda12.8-cudnn9-devel

RUN apt-get update \
    && apt-get install --yes --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/* \
    && node --version \
    && npm --version

COPY requirements.txt /tmp/requirements.txt
RUN python -m pip install \
    --break-system-packages \
    --no-cache-dir \
    -r /tmp/requirements.txt
