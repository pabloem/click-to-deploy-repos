FROM marketplace.gcr.io/google/debian11:latest as builder

RUN apt-get update \
    && apt-get install curl perl build-essential cmake git unzip -y --no-install-recommends
ENV GOLANG_VERSION=1.16
RUN curl -L https://golang.org/dl/go${GOLANG_VERSION}.linux-amd64.tar.gz -o /golang.tar.gz &&tar -xzf /golang.tar.gz -C /usr/local
ENV PATH /usr/local/go/bin:$PATH

ENV TIKV_APP_VERSION=5.3.2

#build tikv
WORKDIR /tikv

RUN curl -L https://github.com/tikv/tikv/archive/refs/tags/v${TIKV_APP_VERSION}.tar.gz -o ./tikv.tar.gz && tar -xzf ./tikv.tar.gz && mv ./tikv-*/* ./

ENV LIBRARY_PATH /usr/local/lib:$LIBRARY_PATH
ENV LD_LIBRARY_PATH /usr/local/lib:$LD_LIBRARY_PATH

RUN curl https://sh.rustup.rs -sSf | sh -s -- --no-modify-path --default-toolchain none -y
ENV PATH /root/.cargo/bin/:$PATH

RUN rustup self update \
  && rustup set profile minimal \
  && rustup default $(cat "rust-toolchain")

ENV CARGO_BUILD_JOBS 32
RUN lscpu

RUN make build_dist_release

#install jq for pd-ctl
RUN cd / && \
    curl -L https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 -o jq && \
    chmod +x jq

#build pd
ENV PD_APP_VERSION=5.3.2

RUN mkdir -p /go/src/github.com/tikv/pd
WORKDIR /go/src/github.com/tikv/pd

RUN curl -L https://github.com/tikv/pd/archive/refs/tags/v${PD_APP_VERSION}.tar.gz -o ./pd.tar.gz && tar -xzf ./pd.tar.gz && mv ./pd-*/* ./
RUN GO111MODULE=on go mod download

RUN make

#build tikvtest
WORKDIR /tikvtest
COPY tikvtest_source/tikvtest.go /tikvtest/tikvtest.go
RUN export GO111MODULE=on \
    && go mod init tikvtest \
    && go get github.com/pingcap/tidb/config@kv-only-4.0 \
    && go get github.com/pingcap/tidb/store/tikv@kv-only-4.0 \
    && go build tikvtest.go

#copy inferno license to /usr/src
RUN cp -r $(find /root/.cargo/registry -name "inferno-*" -type d) /usr/src/

FROM marketplace.gcr.io/google/debian11:latest

ENV PD_ENDPOINT 127.0.0.1:2379

RUN apt-get update \
    && apt-get install --no-install-recommends -yqq curl openssl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

#satisfy CDDL license
COPY --from=builder /usr/src/inferno-* /usr/src/inferno/

COPY --from=builder /tikv/target/release/tikv-server /tikv-server
COPY --from=builder /tikv/target/release/tikv-ctl /tikv-ctl
COPY --from=builder /go/src/github.com/tikv/pd/bin/pd-server /pd-server
COPY --from=builder /go/src/github.com/tikv/pd/bin/pd-ctl /pd-ctl
COPY --from=builder /go/src/github.com/tikv/pd/bin/pd-recover /pd-recover
COPY --from=builder /jq /usr/local/bin/jq
COPY --from=builder /tikvtest/tikvtest /tikvtest

EXPOSE 2379 2380 20160 20180
ENV C2D_RELEASE 5.3.2

CMD ["/tikv-server"]
