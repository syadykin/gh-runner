ARG DOCKER_VERSION=24.0.7
ARG DOCKER_COMPOSE_VERSION=2.23.3
ARG GH_RUNNER_VERSION=2.311.0
ARG DIND_COMMIT=65cfcc28ab37cb75e1560e4b4738719c07c6618e

FROM ubuntu:23.10

ARG DOCKER_VERSION
ARG DOCKER_COMPOSE_VERSION
ARG GH_RUNNER_VERSION
ARG DIND_COMMIT

ENV DOCKER_TLS_CERTDIR=/certs
# exit container if failed
ENV S6_BEHAVIOUR_IF_STAGE2_FAILS=2

RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt update \
    && apt install --no-install-recommends -y \
      ca-certificates openssh-client wget sudo liblttng-ust1 \
      libkrb5-3 zlib1g libssl3 libicu72 groff less unzip curl \
      jq e2fsprogs iptables openssl pigz xz-utils xfsprogs \
      btrfs-progs s6 iproute2 ansible-core ansible python3-pip \
      python3-botocore python3-boto3

RUN addgroup --system dockremap \
	&& useradd -r -g dockremap dockremap \
	&& echo 'dockremap:165536:65536' >> /etc/subuid \
	&& echo 'dockremap:165536:65536' >> /etc/subgid

RUN groupadd --system runner \
    && useradd -mr -g runner --group root -s /bin/bash runner \
    && usermod -aG sudo runner \
    && echo '%sudo ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers

RUN --mount=type=cache,target=/cache \
    url="https://download.docker.com/linux/static/stable/x86_64/docker-${DOCKER_VERSION}.tgz" \
    file="/cache/docker-${DOCKER_VERSION}.tgz" \
    && ([ ! -f $file ] && curl -L $url -o $file || true) \
    && tar --extract --file $file \
      --strip-components 1 \
      --directory /usr/local/bin/ \
      --no-same-owner \
    && docker --version \
    && dockerd --version \
	&& containerd --version \
	&& ctr --version \
	&& runc --version

COPY modprobe.sh /usr/local/bin/modprobe
RUN mkdir -p /certs/client && chmod 1777 -R /certs

RUN --mount=type=cache,target=/cache \
    url="https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" \
    file="/cache/dind-${DIND_COMMIT}" \
	&& ([ ! -f $file ] && curl -L $url -o $file || true) \
    && install -m 0755 $file /usr/local/bin/dind

RUN --mount=type=cache,target=/cache \
    url="https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-linux-x86_64" \
    file="/cache/docker-compose-${DOCKER_COMPOSE_VERSION}" \
    plugin="/usr/local/libexec/docker/cli-plugins/docker-compose" \
    && ([ ! -f $file ] && curl -L $url -o $file || true) \
    && mkdir -p $(dirname $plugin) \
    && install -m755 $file $plugin \
    && ln -sv $plugin /usr/local/bin/

RUN --mount=type=cache,target=/cache \
    url="https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
    file="/cache/awscliv2.zip" \
    && ([ ! -f $file ] && curl -L $url -o $file || true) \
    && unzip $file \
    && ./aws/install \
    && rm -rf ./aws

RUN --mount=type=cache,target=/cache \
    url="https://github.com/actions/runner/releases/download/v${GH_RUNNER_VERSION}/actions-runner-linux-x64-${GH_RUNNER_VERSION}.tar.gz" \
    file="/cache/actions-runner-linux-x64-${GH_RUNNER_VERSION}.tar.gz" \
    && ([ ! -f $file ] && curl -L $url -o $file || true) \
    && tar --extract --file $file --directory /home/runner \
    && chown runner:runner /home/runner -R

ADD s6 /etc/s6

COPY --chmod=0755 dockerd-run.sh /usr/local/bin/dockerd-run.sh

RUN find /etc/s6 -name run -exec chmod +x {} \;

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

CMD [ "s6-svscan", "/etc/s6" ]
