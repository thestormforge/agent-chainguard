# Maintaining agent-chainguard

How the Chainguard images are built and published. This is for maintainers of
this repository, not customers (see [README](README.md) for using the images).

## Layout

```
images/
  workload-agent/Dockerfile         # rebases the agent binary (build-arg AGENT_TAG)
  applier/Dockerfile                # rebases the applier binary (build-arg APPLIER_TAG)
chainguard-values-v2-agent.yaml     # example overrides for the stormforge-agent chart
chainguard-values-v2-applier.yaml   # example overrides for the stormforge-applier chart
.github/workflows/reconcile.yaml    # builds / backfills the Chainguard images
```

## Building locally

```sh
docker build --build-arg AGENT_TAG=2.28.2 -t workload-agent:dev images/workload-agent
docker build --build-arg APPLIER_TAG=2.15.2 -t applier:dev images/applier
```

## Publishing (reconcile)

`reconcile.yaml` builds and publishes to `ghcr.io/thestormforge/agent-chainguard/*`.
It is manual (`workflow_dispatch`) and can only be triggered by someone with
write access — external users cannot run it. It:

- lists the upstream 2.x tags of both images (the agent and applier version
  independently),
- keeps tags at/above the per-stream floor inputs, skipping pre-releases,
- skips anything already published, and builds the rest.

It is idempotent — safe to re-run. Two operational points:

- **When requested**, backfill down to the lowest chart version you intend to
  support, so a customer's repository-only override resolves for every supported
  version.
- **After a new 2.x release**, re-run it so upgrades (whose image tag follows the
  new chart version) resolve to a published image.

It needs list/pull access to `registry.stormforge.io` via registry pull
credentials configured as repository secrets (unless that registry allows
anonymous access).

## Package visibility

GHCR packages default to **private** on first publish, even in a public repo.
Set each package (`workload-agent`, `applier`) to public so customers can pull
anonymously. Repo visibility and package visibility are independent — the repo
can stay private while the packages are published publicly.

## Notes

- The upstream images already run as `nobody` (UID 65534), which matches the
  Chainguard `static` base; the Dockerfiles set `USER 65534:65534` explicitly.
- The applier writes webhook serving certs to a tmpdir
  (`STORMFORGE_MANAGED_WEBHOOK_CERT_DIR`); the Dockerfile carries this env over
  from the upstream image.
