# Maintaining agent-chainguard

How the Chainguard images are built and published. This is for maintainers of
this repository, not customers (see [README](README.md) for using the images).

## Layout

```
images/
  workload-agent/Dockerfile         # rebases the agent binary (build-arg AGENT_TAG)
  applier/Dockerfile                # rebases the applier binary (build-arg APPLIER_TAG)
  node-agent/Dockerfile             # rebases the 3.1+ node-agent (build-arg NODE_AGENT_TAG)
chainguard-values-v2-agent.yaml     # example overrides for the v2 stormforge-agent chart
chainguard-values-v2-applier.yaml   # example overrides for the v2 stormforge-applier chart
chainguard-values-v3.yaml           # example overrides for the 3.4+ stormforge chart
chainguard-values-v3.1-3.3.yaml     # example overrides for the 3.1 to 3.3 stormforge chart
chainguard-values-v3.0.yaml         # example overrides for the 3.0 stormforge chart
.github/workflows/reconcile.yaml    # builds / backfills the Chainguard images
```

## Building locally

```sh
# workload-agent (same Dockerfile for v2 and 3.0 — only the tag differs)
docker build --build-arg AGENT_TAG=2.28.2 -t workload-agent:v2 images/workload-agent
docker build --build-arg AGENT_TAG=3.0.0  -t workload-agent:v3 images/workload-agent
# applier (v2 only)
docker build --build-arg APPLIER_TAG=2.15.2 -t applier:dev images/applier
# node-agent (3.1+, amd64 only)
docker build --platform linux/amd64 --build-arg NODE_AGENT_TAG=3.4.0 -t node-agent:dev images/node-agent
```

## Publishing (reconcile)

`reconcile.yaml` builds and publishes to `ghcr.io/thestormforge/agent-chainguard/*`.
It runs on a schedule and on demand (`workflow_dispatch`), and can only be
triggered by someone with write access — external users cannot run it.

It builds four streams:

- **workload-agent 2.x** and **applier 2.x** (v2 — the agent and applier version
  independently, each with its own floor input),
- **workload-agent 3.x** (same image and Dockerfile, 3.x tags; its floor is
  fixed at 3.0.0, so no input),
- **stormforge-node-agent 3.x** (separate image, first released in 3.1.0, so its
  floor is fixed there; built for linux/amd64 only, matching upstream).

For each stream it lists the upstream tags, keeps the ones at/above the stream's
floor (skipping pre-releases), skips anything already published, and builds the
rest.

It is idempotent — safe to re-run. Two operational points:

- **When requested**, backfill down to the lowest chart version you intend to
  support, so a customer's repository-only override resolves for every supported
  version.
- **After a new release** (2.x or 3.x), re-run it so upgrades (whose image tag
  follows the new chart version) resolve to a published image.

It needs list/pull access to `registry.stormforge.io` via registry pull
credentials configured as repository secrets (unless that registry allows
anonymous access).

## Package visibility

GHCR packages default to **private** on first publish, even in a public repo.
Set each package (`workload-agent`, `applier`, `stormforge-node-agent`) to public so customers can pull
anonymously. Repo visibility and package visibility are independent — the repo
can stay private while the packages are published publicly.

## Notes

- The upstream workload-agent and applier images already run as `nobody`
  (UID 65534), which matches the Chainguard `static` base; the Dockerfiles set
  `USER 65534:65534` explicitly.
- The node-agent is a cgo binary that dlopens NVIDIA libraries, so it uses
  `glibc-dynamic` instead of `static`, carries over the DCGM libs from the
  upstream image, and runs as root like upstream (the chart grants it hostPID
  and SYS_ADMIN). `libnvidia-ml` is injected by the NVIDIA container runtime.
- The applier writes webhook serving certs to a tmpdir
  (`STORMFORGE_MANAGED_WEBHOOK_CERT_DIR`); the Dockerfile carries this env over
  from the upstream image.
