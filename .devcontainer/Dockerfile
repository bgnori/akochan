FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        g++ \
        make \
        libboost-all-dev \
    && rm -rf /var/lib/apt/lists/*
