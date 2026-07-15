# harness-python-yt-dlp-example

A Harness CI pipeline example that builds and tests a fork of [yt-dlp](https://github.com/yt-dlp/yt-dlp), running on a local Kubernetes cluster (Docker Desktop) instead of Harness Cloud.

## Layout

```
.harness/orgs/default/projects/yt_dlp_ci/
  connectors/   GitHub, Kubernetes, and Docker Hub connectors (no secrets)
  pipelines/    the CI pipeline definition
SETUP.md        full setup, start to finish
```

## Setup

See [SETUP.md](./SETUP.md).

## Requirements

Docker Desktop with Kubernetes enabled, the Harness CLI, a GitHub account.
