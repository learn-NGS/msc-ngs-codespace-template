FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

USER root

RUN apt-get update && \
    apt-get install -y \
    wget \
    curl \
    bzip2 \
    ca-certificates \
    git-lfs && \
    rm -rf /var/lib/apt/lists/*

# Install Miniforge (conda + mamba)
RUN wget -q \
    https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh \
    -O /tmp/miniforge.sh && \
    bash /tmp/miniforge.sh -b -p /opt/miniforge3 && \
    rm /tmp/miniforge.sh && \
    /opt/miniforge3/bin/conda clean -afy && \
    chown -R vscode:vscode /opt/miniforge3

ENV PATH="/opt/miniforge3/bin:${PATH}"
ENV MAMBA_ROOT_PREFIX="/opt/miniforge3"

USER vscode

# Initialize Mamba
RUN /opt/miniforge3/bin/mamba shell init \
    -s bash \
    -r /opt/miniforge3 && \
    /opt/miniforge3/bin/conda config --set auto_activate_base false

# Copy the NGS environment definition
COPY --chown=vscode:vscode gvar.yml /tmp/gvar.yml

# Preinstall germline variant-calling environment
RUN /opt/miniforge3/bin/mamba env create \
    -f /tmp/gvar.yml && \
    /opt/miniforge3/bin/conda clean -afy
