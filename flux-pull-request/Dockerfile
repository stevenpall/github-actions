FROM alpine:latest
LABEL maintainer="Steven Pall <mail@stevenpall.ca>"

LABEL "com.github.actions.name"="Flux Deploy Pull Request"
LABEL "com.github.actions.description"="Deploy a pull request via Flux for testing"
LABEL "com.github.actions.icon"="activity"
LABEL "com.github.actions.color"="red"

RUN apk add --no-cache \
    bash \
    ca-certificates \
    curl \
    git \
    jq \
    moreutils

ENV YQ_VERSION 2.2.1
RUN wget -q https://github.com/mikefarah/yq/releases/download/$YQ_VERSION/yq_linux_amd64 -O /usr/local/bin/yq \
    && chmod +x /usr/local/bin/yq

COPY flux-deploy-pull-request /usr/local/bin/flux-deploy-pull-request

ENTRYPOINT ["flux-deploy-pull-request"]
