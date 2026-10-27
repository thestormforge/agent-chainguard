# Rebase the StormForge node-agent (3.1+) onto a Chainguard base image.
#
# Unlike the workload-agent, the node-agent is a cgo binary that dlopens NVIDIA
# libraries at runtime, so it needs glibc (glibc-dynamic, not static):
#   - libnvidia-ml.so.1 is injected by the NVIDIA container runtime.
#   - libdcgm (+ modules) and the DCP profiling backend ship in the upstream image;
#     copy them over unchanged, at the same Debian multiarch path.
# Upstream is linux/amd64 only, so this image is too.
#
# NODE_AGENT_TAG selects which upstream node-agent version to rebase. The
# reconcile workflow passes it as a build-arg; locally it defaults to latest.
ARG NODE_AGENT_TAG=latest

FROM registry.stormforge.io/optimize/stormforge-node-agent:${NODE_AGENT_TAG} AS node-agent

FROM cgr.dev/chainguard/glibc-dynamic:latest
# libdcgm* also picks up the libdcgmmodule* libs that 3.1.x ships (3.2+ doesn't).
COPY --from=node-agent \
    /usr/lib/x86_64-linux-gnu/libdcgm* \
    /usr/lib/x86_64-linux-gnu/libnvperf_dcgm_host.so \
    /usr/lib/x86_64-linux-gnu/
COPY --from=node-agent /stormforge-node-agent /stormforge-node-agent

# Wolfi's loader searches /usr/lib, not the Debian multiarch dir the DCGM libs
# are copied to.
ENV LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu

# The upstream image runs as root (the chart adds hostPID and SYS_ADMIN); keep
# that rather than inheriting the Chainguard nonroot default.
USER 0:0

EXPOSE 9401
ENTRYPOINT ["/stormforge-node-agent"]
