FROM --platform=linux/amd64 debian:12-slim as runner-tools

ENV TARGETOS=linux \
    TARGETARCH=amd64 \
    RUNNER_ARCH=x64 \
    DOCKER_ARCH=x86_64 \
    CONTAINER_HOOK_TYPE=k8s

ARG RUNNER_VERSION=2.322.0
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.2
ARG DOCKER_VERSION=27.5.1
ARG BUILDX_VERSION=0.20.1

RUN apt-get update -y --fix-missing \
    && apt-get upgrade -y \
    && apt-get install curl unzip -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* /var/tmp/*

WORKDIR /actions-runner

# Install Runner
RUN curl -fLo runner.tar.gz https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-${TARGETOS}-${RUNNER_ARCH}-${RUNNER_VERSION}.tar.gz \
    && tar xzf ./runner.tar.gz \
    && rm runner.tar.gz

# Install Container Hooks
RUN curl -fLo runner-container-hooks.zip https://github.com/actions/runner-container-hooks/releases/download/v${RUNNER_CONTAINER_HOOKS_VERSION}/actions-runner-hooks-${CONTAINER_HOOK_TYPE}-${RUNNER_CONTAINER_HOOKS_VERSION}.zip \
    && unzip ./runner-container-hooks.zip -d ./${CONTAINER_HOOK_TYPE} \
    && rm runner-container-hooks.zip

# Install Docker
RUN curl -fLo docker.tgz https://download.docker.com/${TARGETOS}/static/stable/${DOCKER_ARCH}/docker-${DOCKER_VERSION}.tgz \
    && tar zxvf docker.tgz \
    && rm -rf docker.tgz \
    && mkdir -p /usr/local/lib/docker/cli-plugins \
    && curl -fLo /usr/local/lib/docker/cli-plugins/docker-buildx \
        "https://github.com/docker/buildx/releases/download/v${BUILDX_VERSION}/buildx-v${BUILDX_VERSION}.linux-${TARGETARCH}" \
    && chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx


FROM --platform=linux/amd64 debian:12-slim as runner-runtime

ARG DUMB_INIT_VERSION=1.2.5

ENV DEBIAN_FRONTEND=noninteractive \
    RUNNER_MANUALLY_TRAP_SIG=1 \
    ACTIONS_RUNNER_PRINT_LOG_TO_STDOUT=1 \
    ImageOS=debian \
    TARGETOS=linux \
    TARGETARCH=amd64 \
    ARCH=x86_64 \
    RUNNER_UID=1001 \
    RUNNER_GID=123

# 'gpg-agent' and 'software-properties-common' are needed for the 'add-apt-repository' command that follows
RUN apt-get update -y --fix-missing \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends gpg-agent gnupg software-properties-common build-essential sudo lsb-release curl wget jq zip unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# install git latest version
RUN echo "deb http://deb.debian.org/debian bookworm-backports main contrib non-free-firmware" > /etc/apt/sources.list.d/backports.list \
    && apt-get update -y --fix-missing \
    && apt-get upgrade -y \
    && apt-get install -y -t bookworm-backports git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*    

## Install Packages

# Install dumb-init && yq
RUN curl -fLo /usr/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v${DUMB_INIT_VERSION}/dumb-init_${DUMB_INIT_VERSION}_${ARCH} \
    && wget https://github.com/mikefarah/yq/releases/latest/download/yq_${TARGETOS}_${TARGETARCH} -O /usr/bin/yq \
    && chmod +x /usr/bin/dumb-init /usr/bin/yq

# Install AWS CLI v2
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip \
    && aws --version

# Install GitHub CLI
RUN mkdir -p -m 755 /etc/apt/keyrings \
    && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    && cat $out | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && apt-get update --fix-missing -y \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends gh \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && gh --version

# Download latest git-lfs version
RUN curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | bash \
    && apt-get update -y --fix-missing \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends git-lfs \
    && git lfs install --system \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
    
# Set up the runner user
RUN adduser --disabled-password --gecos "" --uid $RUNNER_UID runner \
    && groupadd docker --gid $RUNNER_GID \
    && usermod -aG sudo runner \
    && usermod -aG docker runner \
    && echo "%sudo   ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers \
    && echo "Defaults env_keep += \"DEBIAN_FRONTEND\"" >> /etc/sudoers
    
WORKDIR /home/runner

COPY --chown=runner:docker --from=runner-tools /actions-runner .
COPY --from=runner-tools /usr/local/lib/docker/cli-plugins/docker-buildx /usr/local/lib/docker/cli-plugins/docker-buildx

RUN install -o root -g root -m 755 docker/* /usr/bin/ && rm -rf docker

# 임시 파일 정리 함수
RUN cleanup() { \
        apt-get clean; \
        rm -rf \
            /var/lib/apt/lists/* \
            /tmp/* \
            /var/tmp/* \
            /usr/share/doc \
            /usr/share/man \
            /usr/share/locale; \
    } \
    && cleanup

USER runner
