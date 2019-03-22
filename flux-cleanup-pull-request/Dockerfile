FROM alpine:latest
LABEL maintainer="Steven Pall <mail@stevenpall.ca>"

LABEL "com.github.actions.name"="Flux Cleanup Pull Request"
LABEL "com.github.actions.description"="Cleanup a Flux pull request deployment"
LABEL "com.github.actions.icon"="activity"
LABEL "com.github.actions.color"="red"

RUN apk add --no-cache \
    bash \
    ca-certificates \
    curl \
    git

COPY flux-cleanup-pull-request /usr/local/bin/flux-cleanup-pull-request

ENTRYPOINT ["flux-cleanup-pull-request"]
