FROM debian:10.3-slim

ENV ARGO_VERSION=v2.8.1
ENV IMAGE_OS=linux
ENV IMAGE_ARCH=ppc64le
# Install docker
ENV DOCKER_CHANNEL=stable
ENV DOCKER_VERSION=18.06.3
ENV KUBECTL_VERSION=1.11.0
ENV JQ_VERSION=1.6
ENV GOROOT=/usr/local/go
ENV PATH="$GOROOT/bin:$PATH"

RUN apt-get update && \
  apt-get --no-install-recommends install -y \
  git make apt-utils apt-transport-https ca-certificates \
  curl procps tar mime-support \
  gnupg dirmngr xz-utils libatomic1 nodejs npm \
  wget gcc zip && \
  apt-get clean && \
  rm -rf \
  /var/lib/apt/lists/* \
  /tmp/* \
  /var/tmp/* \
  /usr/share/man \
  /usr/share/doc \
  /usr/share/doc-base

RUN wget -O go.tar.gz https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz; \
    tar -xvf go.tar.gz --directory /usr/local/ && \
    rm go.tar.gz

RUN wget -O docker.tgz https://download.docker.com/linux/static/${DOCKER_CHANNEL}/ppc64le/docker-${DOCKER_VERSION}-ce.tgz; \
    tar --extract --file docker.tgz --strip-components 1 --directory /usr/local/bin/ && \
    rm docker.tgz

ADD hack/recurl.sh .
RUN ./recurl.sh /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v${KUBECTL_VERSION}/bin/linux/${IMAGE_ARCH}/kubectl
RUN ./recurl.sh /usr/local/bin/jq https://github.com/stedolan/jq/releases/download/jq-${JQ_VERSION}/jq-linux64
RUN rm recurl.sh
COPY hack/ssh_known_hosts /etc/ssh/ssh_known_hosts

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN  echo "deb https://dl.yarnpkg.com/debian/ stable main" > /etc/apt/sources.list.d/yarn.list && \
  apt-get update && apt-get --no-install-recommends install -y \
  yarn build-essential

ADD ["ui", "ui"]
ADD ["api", "api"]
# cli image
RUN mkdir -p ui/dist
RUN yarn --cwd ui install
RUN yarn --cwd ui build

# Perform the build
WORKDIR /go/src/github.com/argoproj/argo
# COPY . .
RUN git clone https://github.com/harshad16/argo.git .
# check we can use Git
RUN git rev-parse HEAD

# controller image
RUN make dist/workflow-controller-linux-${IMAGE_ARCH}
RUN ["sh", "-c", "./dist/workflow-controller-linux-${IMAGE_ARCH} version | grep clean"]

# executor image
RUN make dist/argoexec-linux-${IMAGE_ARCH}
RUN ["sh", "-c", "./dist/argoexec-linux-${IMAGE_ARCH} version | grep clean"]

# stop make from trying to re-build this without yarn installed
RUN touch ui/dist/node_modules.marker
RUN touch ui/dist/app/index.html
RUN make argo-server.crt argo-server.key dist/argo-linux-${IMAGE_ARCH}
RUN ["sh", "-c", "./dist/argo-linux-${IMAGE_ARCH} version | grep clean"]

COPY /go/src/github.com/argoproj/argo/argo-server.crt argo-server.crt
COPY /go/src/github.com/argoproj/argo/argo-server.key argo-server.key
COPY /go/src/github.com/argoproj/argo/dist/argo-linux-* /bin/argo
ENTRYPOINT [ "argo" ]
