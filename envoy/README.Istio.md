# Envoy binary for Istio sidecar proxy

The integration of Cilium and Istio requires building artifacts from
several repositories in order to build Docker images.  Some of those
artifacts require changes that have not yet been merged upstream.

This document provides the instructions to build the Cilium-specific
Istio images.

## Build an Envoy binary with both Cilium and Istio filters

    mkdir -p ${GOPATH}/src/github.com/cilium
    cd ${GOPATH}/src/github.com/cilium
    git clone git@github.com:cilium/cilium.git
    cd cilium/envoy
    git checkout build-envoy-transparent-istio
    make istio-envoy

Make the Envoy binary ready to package into Istio's Docker images, as
a version named `transparent`.

    mkdir -p ${GOPATH}/out/linux_amd64/debug/
    mkdir -p ${GOPATH}/out/linux_amd64/release/
    cp ${GOPATH}/src/github.com/cilium/cilium/envoy/istio-envoy ${GOPATH}/out/linux_amd64/debug/envoy-debug-cilium
    cp ${GOPATH}/src/github.com/cilium/cilium/envoy/istio-envoy ${GOPATH}/out/linux_amd64/release/envoy-cilium

## Build the Istio binaries

Build the Istio binaries, especially a `pilot-discovery` modified to
configure Envoy listeners as "transparent" (using iptables TPROXY) and
to configure Cilium filters in every HTTP filter chain.  This work is
still being developed in Cilium's `inject-cilium-filters` branch.

    mkdir -p ${GOPATH}/src/istio.io
    cd ${GOPATH}/src/istio.io
    git clone git@github.com:cilium/istio.git
    git checkout inject-cilium-filters
    git submodule sync
    git submodule update --init --recursive --remote
    git submodule update --force --checkout
    ISTIO_ENVOY_VERSION=cilium make build

## Build the IstioDocker images

4 images need to be built: `cilium/istio_pilot`,
`cilium/istio_proxy_init`, `cilium/istio_proxyv2` and
`cilium/istio_proxy_debugv2`.

Note that we are building Istio's experimental "v2" proxy images,
which use Envoy's V2 API, as required to configure the Cilium filters.

    ISTIO_ENVOY_VERSION=cilium HUB=cilium TAG=0.7.0 make docker.pilot docker.proxy_init docker.proxyv2 docker.proxy_debugv2

## (Optional) Check that the images are consistent

### `cilium/istio_proxy_init`

    docker run --rm -it --cap-add=NET_ADMIN --entrypoint /bin/bash cilium/istio_proxy_init:0.7.0

In the container, run the iptables configuration script with various
combinations of parameters, for example:


    /usr/local/bin/istio-iptables.sh -p 15001 -u 1337 -t -b '*'
    /usr/local/bin/istio-iptables.sh -p 15001 -u 1337 -t -b 1234,5678
    /usr/local/bin/istio-iptables.sh -p 15001 -u 1337 -t -b '*' -d 1234,5678

For each combination, check the iptables and routing tables and rules:

    iptables -v -t nat -L
    iptables -v -t mangle -L
    ip rule
    ip route show table 100

### `cilium/istio_proxyv2`

    docker run --rm -it --entrypoint /bin/bash cilium/istio_proxyv2:0.7.0

Check that the binaries have the right size, mode, owner, and group.

    stat /usr/local/bin/{envoy,pilot-agent}

Check that the xds-grpc-cilium cluster is properly defined, and is an
Envoy V2 API bootstrap (not V1).

    cat /var/lib/istio/envoy/envoy_bootstrap_tmpl.json

### `cilium/istio_proxy_debugv2`

    docker run --rm -it --entrypoint /bin/bash cilium/istio_proxy_debugv2:0.7.0

Check that the binaries have the right size, mode, owner, and group.

    stat /usr/local/bin/{envoy,pilot-agent}

Check that the xds-grpc-cilium cluster is properly defined, and is an
Envoy V2 API bootstrap (not V1).

    cat /var/lib/istio/envoy/envoy_bootstrap_tmpl.json

## Push the Docker images to Docker Hub

    docker login -u ...
    docker image push cilium/istio_pilot:0.7.0
    docker image push cilium/istio_proxy_init:0.7.0
    docker image push cilium/istio_proxyv2:0.7.0
    docker image push cilium/istio_proxy_debugv2:0.7.0
