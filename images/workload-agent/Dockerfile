# Rebase the StormForge workload-agent onto a Chainguard base image.
#
# This is intentionally minimal: copy the already-compiled binary out of the
# published StormForge image and drop it onto cgr.dev/chainguard/static.
# Nothing is compiled here.
#
# In v2 this image backs the agent (the applier is a separate image — see
# images/applier). In the 3.0 chart the same image backs both components.
#
# AGENT_TAG selects which upstream agent image to rebase (e.g. "2.25.0"). The
# workflows pass it as a build-arg; locally it defaults to latest.
ARG AGENT_TAG=latest

FROM registry.stormforge.io/optimize/workload-agent:${AGENT_TAG} AS agent

FROM cgr.dev/chainguard/static:latest
COPY --from=agent /stormforge-agent /stormforge-agent

# The upstream image runs as nobody (65534); set it explicitly so the user is
# unambiguous regardless of the Chainguard base default.
USER 65534:65534

ENTRYPOINT ["/stormforge-agent"]
