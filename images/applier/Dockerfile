# Rebase the StormForge applier onto a Chainguard base image.
#
# Used for pre-3.0 (v2) StormForge, where the applier is a SEPARATE image from
# the workload-agent (in 3.0 the applier folds into the workload-agent image and
# this Dockerfile is not needed).
#
# Like the agent image, this is a minimal rebase: copy the already-compiled
# applier binary onto cgr.dev/chainguard/static. Nothing is recompiled.
#
# APPLIER_TAG selects which upstream applier version to rebase. The reconcile
# workflow passes it as a build-arg; locally it defaults to latest.
ARG APPLIER_TAG=latest

FROM registry.stormforge.io/optimize/applier:${APPLIER_TAG} AS applier

FROM cgr.dev/chainguard/static:latest
COPY --from=applier /applier /applier

# The applier writes its webhook serving certs to disk; keep the upstream
# default path (a writable tmpdir).
ENV STORMFORGE_MANAGED_WEBHOOK_CERT_DIR=/tmp/k8s-webhook-server/serving-certs

# The upstream image runs as nobody (65534); set it explicitly.
USER 65534:65534

ENTRYPOINT ["/applier"]
CMD ["run"]
