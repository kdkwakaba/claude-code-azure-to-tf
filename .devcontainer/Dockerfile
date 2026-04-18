ARG VARIANT="ubuntu-24.04"
FROM mcr.microsoft.com/devcontainers/base:${VARIANT}

ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="/home/vscode/.local/bin:${PATH}"

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        gnupg \
        jq \
        ripgrep \
        unzip \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://packages.microsoft.com/keys/microsoft.asc \
        | gpg --dearmor -o /etc/apt/keyrings/microsoft.gpg \
    && chmod go+r /etc/apt/keyrings/microsoft.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/microsoft.gpg] https://packages.microsoft.com/repos/azure-cli/ $(. /etc/os-release && echo ${VERSION_CODENAME}) main" \
        > /etc/apt/sources.list.d/azure-cli.list \
    && curl -fsSL https://apt.releases.hashicorp.com/gpg \
        | gpg --dearmor -o /etc/apt/keyrings/hashicorp.gpg \
    && chmod go+r /etc/apt/keyrings/hashicorp.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(. /etc/os-release && echo ${VERSION_CODENAME}) main" \
        > /etc/apt/sources.list.d/hashicorp.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        azure-cli \
        terraform \
    && rm -rf /var/lib/apt/lists/* \
    && AZTFEXPORT_VERSION=$(curl -fsSL https://api.github.com/repos/Azure/aztfexport/releases/latest \
        | jq -r '.tag_name') \
    && ARCH=$(dpkg --print-architecture) \
    && curl -fsSL "https://github.com/Azure/aztfexport/releases/download/${AZTFEXPORT_VERSION}/aztfexport_${AZTFEXPORT_VERSION}_linux_${ARCH}.zip" \
        -o /tmp/aztfexport.zip \
    && unzip /tmp/aztfexport.zip aztfexport -d /usr/local/bin/ \
    && chmod +x /usr/local/bin/aztfexport \
    && rm /tmp/aztfexport.zip

USER vscode

RUN curl -fsSL https://claude.ai/install.sh | bash -s stable
