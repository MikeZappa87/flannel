FROM --platform=$BUILDPLATFORM tonistiigi/xx AS xx

FROM --platform=$BUILDPLATFORM golang:alpine AS build
# copy xx scripts to your build stage
COPY --from=xx / /
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TAG
RUN apk --no-cache add bash make gcc musl-dev git linux-headers git
COPY Makefile go.mod go.sum *.go /build/
COPY pkg /build/pkg
WORKDIR /build
RUN mkdir dist
RUN xx-info env
RUN make deps
RUN export GOOS=$(xx-info os) &&\
    export GOARCH=$(xx-info arch) &&\
    export ARCH=$(xx-info arch) &&\
    make dist/flanneld
RUN git clone https://github.com/kubernetes-sigs/iptables-wrappers.git /iptables-wrapper
WORKDIR /iptables-wrapper
RUN git checkout 5792812d9e5a5bb7f22d79d557bbfeece253343d
RUN export GOOS=$(xx-info os) &&\
    export GOARCH=$(xx-info arch) &&\
    export ARCH=$(xx-info arch) &&\
    make build

ARG CNI_PLUGINS_VERSION="v1.3.0"
ARG CNI_PLUGINS_CLONE_URL="https://github.com/containernetworking/plugins"
RUN git clone --filter=tree:0 "${CNI_PLUGINS_CLONE_URL}" /cni-plugins 
WORKDIR /cni-plugins
RUN cd /cni-plugins \
    && git checkout "${CNI_PLUGINS_VERSION}" \
    && CGO_ENABLED=0 ./build_linux.sh && mkdir -p /opt/cni/bin \
    && cp -a ./bin/. /opt/cni/bin/

RUN git clone https://github.com/flannel-io/cni-plugin.git /cni-plugin
WORKDIR /cni-plugin    
RUN cd /cni-plugin \
    && make build_linux && cp -a ./dist/flannel-amd64 /opt/cni/bin/flannel

FROM alpine:20230901
RUN apk update && apk upgrade
RUN apk add --no-cache iproute2 net-tools ca-certificates iptables strongswan iptables-legacy && update-ca-certificates
RUN apk add wireguard-tools --no-cache --repository http://dl-cdn.alpinelinux.org/alpine/edge/community
COPY --from=build /build/dist/flanneld /opt/bin/flanneld
COPY dist/mk-docker-opts.sh /opt/bin/
COPY --from=build /iptables-wrapper/iptables-wrapper-installer.sh /
COPY --from=build /iptables-wrapper/bin/iptables-wrapper /
COPY --from=build /opt/cni/bin /opt/cni/bin
RUN /iptables-wrapper-installer.sh --no-sanity-check


ENTRYPOINT ["/opt/bin/flanneld"]

