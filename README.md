# agent-chainguard

Chainguard-based container images for StormForge, for customers who require
their workloads to run on Chainguard base images. Each image copies the
already-compiled StormForge binary onto a Chainguard base — the same software,
rebased; nothing is recompiled.

> Scope: **pre-3.0 (v2) StormForge** — the `stormforge-agent` and
> `stormforge-applier` charts.

## Images

| Image | Source | Used by |
|-------|--------|---------|
| workload-agent | `registry.stormforge.io/optimize/workload-agent` | `stormforge-agent` chart (`workload.image`) |
| applier | `registry.stormforge.io/optimize/applier` | `stormforge-applier` chart (`image`) |

Published to `ghcr.io/thestormforge/agent-chainguard/*`, **tag-for-tag** with
upstream (e.g. `.../applier:2.15.2` rebases
`registry.stormforge.io/optimize/applier:2.15.2`).

## Using the images

v2 installs the agent and applier as two separate charts — **agent first, then
applier**. Override each chart's image **repository**:

```sh
# agent
helm install stormforge-agent oci://registry.stormforge.io/library/stormforge-agent \
  -f my-values.yaml \
  --set workload.image.repository=ghcr.io/thestormforge/agent-chainguard/workload-agent

# applier
helm install stormforge-applier oci://registry.stormforge.io/library/stormforge-applier \
  -f my-values.yaml \
  --set image.repository=ghcr.io/thestormforge/agent-chainguard/applier
```

The tag is inherited from the chart version and published tag-for-tag, so
overriding only the repository resolves the correct version and keeps working
across upgrades. Pin an exact version with `--set workload.image.tag=<v>` /
`--set image.tag=<v>` if needed. Re-supply the overrides on every `helm upgrade`
— Helm does not remember them between commands.

### Prometheus

The agent chart also runs a Prometheus-based metrics forwarder (a third-party
image) that these images do not include. To run it on Chainguard, point the
forwarder at your own Chainguard Prometheus:

```yaml
prom:
  image:
    repository: <your-chainguard-prometheus-repo>
    tag: <version matching the chart's default prom.image.tag>
```

> Use a **shelled** Prometheus variant — the forwarder's init container runs
> `/bin/sh` to template its config, so a shell-less image fails to start.

---

Maintaining this repo (building/publishing the images): see
[CONTRIBUTING.md](CONTRIBUTING.md).
