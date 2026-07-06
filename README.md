# agent-chainguard

Chainguard-based container images for StormForge, for customers who require
their workloads to run on Chainguard base images.

This is deliberately lightweight: it does not change the core product. Each
image is a minimal two-stage Dockerfile that copies the already-compiled
StormForge binary out of the published image and drops it onto a Chainguard
base — nothing is recompiled. The result is a StormForge image built on a
Chainguard base (not an image published in Chainguard's catalog).

> Scope: **pre-3.0 (v2) StormForge** — the `stormforge-agent` and
> `stormforge-applier` charts. (In 3.0 the applier folds into the agent image;
> support can be added when needed.)

## Images

| Image | Source | Used by |
|-------|--------|---------|
| workload-agent | `registry.stormforge.io/optimize/workload-agent` | v2 `stormforge-agent` chart (`workload.image`) |
| applier | `registry.stormforge.io/optimize/applier` | v2 `stormforge-applier` chart (`image`) |

Both are static Go binaries, so they run on `cgr.dev/chainguard/static` — a
drop-in replacement for the `gcr.io/distroless/static` base the standard images
already use. Images are published **tag-for-tag** with upstream (e.g. upstream
`2.25.0` → `ghcr.io/thestormforge/agent-chainguard/workload-agent:2.25.0`), so
charts resolve the correct version without extra configuration.

## Layout

```
images/
  workload-agent/Dockerfile         # rebases the agent binary (build-arg AGENT_TAG)
  applier/Dockerfile                # rebases the applier binary (build-arg APPLIER_TAG)
chainguard-values-v2-agent.yaml     # overrides for the stormforge-agent chart
chainguard-values-v2-applier.yaml   # overrides for the stormforge-applier chart
.github/workflows/
  build.yaml                        # single-image build (push to main / dispatch)
  reconcile.yaml                    # idempotent backfill of all 2.x versions
```

## Using the images with Helm

The two v2 charts are installed separately, so each gets its own override file.
Add the matching file to your usual install:

```sh
# agent
helm install ... -f my-values.yaml -f chainguard-values-v2-agent.yaml

# applier
helm install ... -f my-values.yaml -f chainguard-values-v2-applier.yaml
```

Only the image **repository** is overridden; the packaged charts pin the image
tag they were tested with, and the tag-for-tag publishing means the correct
Chainguard image resolves automatically. Customers who mirror images into an
internal registry can mirror these images and override the repositories to
their internal paths in the same way.

## Prometheus (the metrics forwarder)

The v2 agent chart also runs a Prometheus-based metrics forwarder (a
third-party image). This repo does **not** rebase or host Prometheus. Customers
who standardise on Chainguard point the forwarder at their own Chainguard
Prometheus:

```yaml
prom:
  image:
    repository: <your-chainguard-prometheus-repo>
    tag: <version matching the chart's default prom.image.tag>
```

> **Gotcha:** use a **shelled** Prometheus variant. The forwarder's init
> container runs `/bin/sh` + `sed` (using the Prometheus image) to template its
> config, so a distroless/shell-less image fails to start with `sh: not found`.

## Building locally

```sh
docker build --build-arg AGENT_TAG=2.25.0 \
  -t workload-agent:dev images/workload-agent

docker build --build-arg APPLIER_TAG=2.15.2 \
  -t applier:dev images/applier
```

## Publishing

Two workflows, both pushing to `ghcr.io/thestormforge/agent-chainguard/*`:

- **`build.yaml`** — builds the workload-agent image. Runs on push to `main`
  (against the upstream `latest` tag) or via manual dispatch with a specific
  `agent_tag`.
- **`reconcile.yaml`** — manual one-off backfill. Discovers all upstream 2.x
  versions of both images (above per-stream floor inputs; the agent and applier
  version independently), skips anything already published, and builds the
  rest. Idempotent — safe to re-run; it only builds what is missing.

The reconcile workflow needs list/pull access to `registry.stormforge.io`
(secrets `STORMFORGE_REGISTRY_USERNAME` / `STORMFORGE_REGISTRY_PASSWORD`)
unless the registry allows anonymous access.

> **GHCR visibility:** packages created by a first push default to **private**,
> even in a public repository. After the first publish, set each package
> (`workload-agent`, `applier`) to public in its package settings so customers
> can pull anonymously.

## Notes

- The upstream images already run as `nobody` (UID 65534), which matches the
  Chainguard `static` base; the Dockerfiles set `USER 65534:65534` explicitly.
- The applier writes webhook serving certs to a tmpdir
  (`STORMFORGE_MANAGED_WEBHOOK_CERT_DIR`); the Dockerfile carries this env over
  from the upstream image.
