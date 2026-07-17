# agent-chainguard

Chainguard-based container images for StormForge, for customers who require
their workloads to run on Chainguard base images. Each image copies the
already-compiled StormForge binary onto a Chainguard base — the same software,
rebased; nothing is recompiled.

> Scope: **StormForge v2 and 3.0.** v2 uses the separate `stormforge-agent` and
> `stormforge-applier` charts (two images). 3.0 uses the single `stormforge`
> chart, where one `workload-agent` image backs both the agent and applier.

## Images

| Image | Source | Used by |
|-------|--------|---------|
| workload-agent | `registry.stormforge.io/optimize/workload-agent` | v2 `stormforge-agent` chart (`workload.image`); 3.0 `stormforge` chart (both `agent.image` and `applier.image`) |
| applier | `registry.stormforge.io/optimize/applier` | v2 `stormforge-applier` chart (`image`) — **v2 only**; 3.0 has no separate applier image |

Published to `ghcr.io/thestormforge/agent-chainguard/*`, **tag-for-tag** with
upstream (e.g. `.../workload-agent:3.0.0` rebases
`registry.stormforge.io/optimize/workload-agent:3.0.0`).

## Using the images — 3.0

3.0 installs everything from the single `stormforge` chart. Point it at the
Chainguard images with `--set` flags on your normal install — no values file
needed. The chart has no global image setting, so set **both** the agent and
applier (they share the one `workload-agent` image):

```sh
helm install stormforge oci://registry.stormforge.io/library/stormforge \
  --namespace stormforge-system --create-namespace \
  --set agent.image.repository=ghcr.io/thestormforge/agent-chainguard/workload-agent \
  --set applier.image.repository=ghcr.io/thestormforge/agent-chainguard/workload-agent
```

Keep your usual `clusterName` / `authorization` values as well. Add
`--set 'imagePullSecrets[0].name=<secret>'` only if the GHCR package is private.

You don't need to set the tag — it's inherited from the chart version and
published tag-for-tag, so the right version resolves automatically and keeps
working across upgrades. Pin one with `--set agent.image.tag=<v>` /
`--set applier.image.tag=<v>` if you want.

Prefer a file? `chainguard-values-v3.yaml` bundles the same two overrides (handy
when layering more customisation) — add it with `-f chainguard-values-v3.yaml`.

## Using the images — v2

v2 installs the agent and applier as two separate charts — **agent first, then
applier**. Add an image **repository** override to your normal install for each
chart (keeping your usual cluster name, credentials, and other values):

```sh
# agent
helm install stormforge-agent oci://registry.stormforge.io/library/stormforge-agent \
  --set workload.image.repository=ghcr.io/thestormforge/agent-chainguard/workload-agent

# applier
helm install stormforge-applier oci://registry.stormforge.io/library/stormforge-applier \
  --set image.repository=ghcr.io/thestormforge/agent-chainguard/applier
```

The tag is inherited from the chart version and published tag-for-tag, so
overriding only the repository resolves the correct version and keeps working
across upgrades. Pin an exact version with `--set workload.image.tag=<v>` /
`--set image.tag=<v>` if needed. Re-supply the overrides on every `helm upgrade`
— Helm does not remember them between commands. Example files
`chainguard-values-v2-agent.yaml` / `chainguard-values-v2-applier.yaml` bundle
these overrides if you prefer `-f`.

### Prometheus

Both versions also run a Prometheus-based metrics component (a third-party image)
that these images do not include. To run it on Chainguard, point it at your own
Chainguard Prometheus — `prom.image` in v2, `exporter.image` in 3.0:

```yaml
# v2 example
prom:
  image:
    repository: <your-chainguard-prometheus-repo>
    tag: <version matching the chart default>
```

Two things to get right, or the Prometheus pod won't start:
- Use a **shelled** Prometheus variant — the init container runs `/bin/sh` to
  template its config, so a shell-less image fails with `sh: not found`.
- Set **`WORKDIR /prometheus`** on the image, or it fails with
  `mkdir /data-agent: permission denied`. Wrap the Chainguard Prometheus image
  with a two-line Dockerfile that adds `WORKDIR /prometheus`.

---

Maintaining this repo (building/publishing the images): see
[CONTRIBUTING.md](CONTRIBUTING.md).
