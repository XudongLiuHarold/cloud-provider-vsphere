################################################################################
##                               BUILD ARGS                                   ##
################################################################################
# The golang image is used to create the project's module and build caches
# and is also the image on which this image is based.
ARG GOLANG_IMAGE=golang:1.22

# The image from which the Terraform project used to turn up a K8s cluster is
# copied, as well as several programs.
ARG SK8E2E_IMAGE=gcr.io/kubernetes-conformance-testing/sk8e2e:v0.2.1-29-g1d251af

################################################################################
##                            GO MOD CACHE STAGE                              ##
################################################################################
# Create a Go module cache.
FROM ${GOLANG_IMAGE} as mod-cache
WORKDIR /build
COPY go.mod go.sum ./
COPY pkg ./pkg/
COPY cmd ./cmd/
ARG GOOS
ARG GOARCH
ARG GOPROXY
ENV GOOS=${GOOS:-linux} GOARCH=${GOARCH:-amd64}
ENV GOPROXY ${GOPROXY:-https://proxy.golang.org}
RUN go mod download && go mod verify

################################################################################
##                           GO BUILD CACHE STAGE                             ##
################################################################################
# Create a Go build cache. Please note the reason the Makefile is not used and
# "go build" is invoked directly is to avoid having to rebuild this stage as a
# result of the Makefile changing.
FROM ${GOLANG_IMAGE} as build-cache
WORKDIR /build
COPY --from=mod-cache /go/pkg/mod /go/pkg/mod/
COPY go.mod go.sum hack/make/ldflags.txt ./
COPY pkg ./pkg/
COPY cmd ./cmd/
ARG GOOS
ARG GOARCH
ENV CGO_ENABLED=0 GOOS=${GOOS:-linux} GOARCH=${GOARCH:-amd64}
RUN LDFLAGS=$(cat ldflags.txt) && \
    go build -ldflags "${LDFLAGS}" ./cmd/vsphere-cloud-controller-manager && \
    go build -ldflags "${LDFLAGS}" ./cmd/vcpctl

################################################################################
##                              SK8E2E STAGE                                  ##
################################################################################
FROM ${SK8E2E_IMAGE} as sk8e2e
RUN /google-cloud-sdk/bin/gcloud components update

################################################################################
##                               KIND STAGE                                   ##
################################################################################
FROM ${GOLANG_IMAGE} as kind
RUN cd / && GO111MODULE="on" go get -u sigs.k8s.io/kind@v0.23.0

################################################################################
##                               MAIN STAGE                                   ##
################################################################################
FROM ${GOLANG_IMAGE}
LABEL "maintainer"="Andrew Kutz <akutz@vmware.com>"

################################################################################
##                            CONFIGURE LOCALE                                ##
################################################################################
# Set the locale so that the gist command is happy.
ENV LANG=en_US.UTF-8 LC_ALL=C.UTF-8

################################################################################
##                             PACKAGE UPDATES                                ##
################################################################################
# Install the dependencies. The list is a uniion of the dependencies required
# by the following images:
#   * https://github.com/vmware/simple-k8s-test-env/blob/master/e2e/Dockerfile
#   * https://github.com/kubernetes/test-infra/blob/master/images/bootstrap/Dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      git \
      jq \
      locales \
      mercurial \
      python3 \
      python3-pip \
      ruby \
      tar \
      unzip \
      zip && \
    rm -rf /var/cache/apt/* /var/lib/apt/lists/* && \
    pip3 install setuptools wheel --upgrade && \
    pip3 install awscli --upgrade && \
    gem install gist

################################################################################
##                             DOCKER-IN-DOCKER                               ##
################################################################################
# Again, copied from test-infra's bootstrap image:
# https://github.com/kubernetes/test-infra/blob/master/images/bootstrap/Dockerfile

# Install Docker deps, some of these are already installed in the image but
# that's fine since they won't re-install and we can reuse the code below
# for another image someday.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      apt-transport-https \
      ca-certificates \
      curl \
      gnupg2 \
      software-properties-common \
      lsb-release && \
    rm -rf /var/cache/apt/* /var/lib/apt/lists/*

# Add the Docker apt-repository
RUN curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "${ID}")/gpg | \
    gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
      $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
# TODO(bentheelder): the `sed` is a bit of a hack, look into alternatives.
# Why this exists: `docker service start` on debian runs a `cgroupfs_mount` method,
# We're already inside docker though so we can be sure these are already mounted.
# Trying to remount these makes for a very noisy error block in the beginning of
# the pod logs, so we just comment out the call to it... :shrug:
# TODO(benthelder): update docker version. This is pinned because of
# https://github.com/kubernetes/test-infra/issues/6187
RUN apt-get update && \
    apt-get install -y --no-install-recommends docker-ce=5:20.10.* && \
    rm -rf /var/cache/apt/* /var/lib/apt/lists/* && \
    sed -i 's/cgroupfs_mount$/#cgroupfs_mount\n/' /etc/init.d/docker

# Move Docker's storage location
RUN echo 'DOCKER_OPTS="${DOCKER_OPTS} --data-root=/docker-graph"' | \
    tee --append /etc/default/docker

# NOTE this should be mounted and persisted as a volume ideally (!)
# We will make a fallback one now just in case
RUN mkdir /docker-graph

# Setting this environment variable is an easy way for processes running
# in the container to know DinD is enabled.
ENV DOCKER_IN_DOCKER_ENABLED=true

################################################################################
##                          COPY FROM SK8E2E                                  ##
################################################################################
COPY --from=sk8e2e /tf                       /tf/
COPY --from=sk8e2e /google-cloud-sdk         /google-cloud-sdk/
COPY --from=sk8e2e /usr/local/bin/govc       /usr/local/bin/
COPY --from=sk8e2e /usr/local/bin/keepalive  /usr/local/bin/
COPY --from=sk8e2e /usr/local/bin/kubectl    /usr/local/bin/
COPY --from=sk8e2e /usr/local/bin/sonobuoy   /usr/local/bin/
COPY --from=sk8e2e /usr/local/bin/terraform  /usr/local/bin/

################################################################################
##                       CONFIGURE GOOGLE CLOUD SDK                           ##
################################################################################
# Update the PATH to include the Google Cloud SDK and disable its prompts and
# update the gcloud components.
ENV PATH="/google-cloud-sdk/bin:${PATH}" CLOUDSDK_CORE_DISABLE_PROMPTS=1

################################################################################
##                             INSTALL KIND                                   ##
################################################################################
COPY --from=kind /go/bin/kind /usr/local/bin/

################################################################################
##                         PRIME GO MOD & BUILD CACHES                        ##
################################################################################
COPY --from=mod-cache   /go/pkg/mod           /go/pkg/mod/
COPY --from=build-cache /root/.cache/go-build /root/.cache/go-build/
RUN  mkdir -p /home/prow/go/pkg && ln -s /go/pkg/mod /home/prow/go/pkg/mod

################################################################################
##                           ADD LOCAL SOURCES                                ##
################################################################################
# Copy the sources into the project's traditional Gopath location in the
# image. It's possible to bind mount up-to-date sources over the ones in
# the image when the latter is run as a container.
WORKDIR /go/src/k8s.io/cloud-provider-vsphere/
COPY . ./
