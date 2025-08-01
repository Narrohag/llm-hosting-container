# Fetch and extract the TGI sources
FROM alpine AS tgi
RUN mkdir -p /tgi
ADD https://github.com/huggingface/text-generation-inference/archive/refs/tags/v3.0.0.tar.gz /tgi/sources.tar.gz
RUN tar -C /tgi -xf /tgi/sources.tar.gz --strip-components=1

# Fetch also the optimum-neuron sources that contain modified TGI sources
FROM alpine AS optimum-neuron
RUN mkdir -p /optimum-neuron
ADD https://github.com/huggingface/optimum-neuron/archive/refs/tags/v0.0.28.tar.gz /optimum-neuron/sources.tar.gz
RUN tar -C /optimum-neuron -xf /optimum-neuron/sources.tar.gz --strip-components=1

# Build cargo components (adapted from TGI original Dockerfile)
# Note: we cannot use the cargo-chef base image as it uses python 3.11
FROM ubuntu:22.04 AS chef

RUN apt-get update -y \
 && apt-get install -y --no-install-recommends \
    curl ca-certificates build-essential \
    && apt-get upgrade -y linux-libc-dev \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- --default-toolchain 1.80.1 --profile minimal -y
ENV PATH="/root/.cargo/bin:${PATH}"
RUN cargo install cargo-chef --locked

WORKDIR /usr/src

ARG CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse

FROM chef AS planner
COPY --from=optimum-neuron /optimum-neuron/text-generation-inference/Cargo.toml Cargo.toml
COPY --from=tgi /tgi/Cargo.lock Cargo.lock
COPY --from=tgi /tgi/rust-toolchain.toml rust-toolchain.toml
COPY --from=tgi /tgi/proto proto
COPY --from=tgi /tgi/router router
COPY --from=tgi /tgi/backends backends
COPY --from=tgi /tgi/launcher launcher
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder

RUN apt-get update -y \
 && apt-get install -y --no-install-recommends \
    unzip python3-dev libssl-dev pkg-config \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

RUN PROTOC_ZIP=protoc-21.12-linux-x86_64.zip && \
    curl -OL https://github.com/protocolbuffers/protobuf/releases/download/v21.12/$PROTOC_ZIP && \
    unzip -o $PROTOC_ZIP -d /usr/local bin/protoc && \
    unzip -o $PROTOC_ZIP -d /usr/local 'include/*' && \
    rm -f $PROTOC_ZIP

COPY --from=optimum-neuron /optimum-neuron/text-generation-inference/Cargo.toml Cargo.toml
COPY --from=planner /usr/src/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json

COPY --from=tgi /tgi/Cargo.lock Cargo.lock
COPY --from=tgi /tgi/rust-toolchain.toml rust-toolchain.toml
COPY --from=tgi /tgi/proto proto
COPY --from=tgi /tgi/router router
COPY --from=tgi /tgi/backends backends
COPY --from=tgi /tgi/launcher launcher
# Remove this line once TGI has fixed the conflict
RUN cargo update ureq --precise 2.9.7
RUN cargo build --release

# Python base image
FROM ubuntu:22.04 AS base

RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
    python3-pip \
    python3-setuptools \
    python-is-python3 \
    && apt-get upgrade -y linux-libc-dev \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean
RUN pip3 --no-cache-dir install --upgrade pip

# Python server build image
FROM base AS pyserver

RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
    make \
    python3-venv \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

RUN install -d /pyserver
WORKDIR /pyserver
COPY --from=optimum-neuron /optimum-neuron/text-generation-inference/server server
COPY --from=tgi /tgi/proto proto
RUN pip3 install -r server/build-requirements.txt
RUN VERBOSE=1 BUILDDIR=/pyserver/build PROTODIR=/pyserver/proto make -C server gen-server

# Neuron base image (used for deployment)
FROM base AS neuron

# Install system prerequisites
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
    gnupg2 \
    wget \
    python3-dev \
    libexpat1 \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

RUN echo "deb https://apt.repos.neuron.amazonaws.com jammy main" > /etc/apt/sources.list.d/neuron.list
RUN wget -qO - https://apt.repos.neuron.amazonaws.com/GPG-PUB-KEY-AMAZON-AWS-NEURON.PUB | apt-key add -

# Install neuronx packages
RUN apt-get update -y \
    && apt-get install -y --no-install-recommends \
    aws-neuronx-dkms=2.18.20.0 \
    aws-neuronx-collectives=2.22.33.0-d2128d1aa \
    aws-neuronx-runtime-lib=2.22.19.0-5856c0b42 \
    aws-neuronx-tools=2.19.0.0 \
    libxml2 \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

ENV PATH="/opt/bin/:/opt/aws/neuron/bin:${PATH}"

RUN pip3 install \
    neuronx-cc==2.15.143.0 \
    torch-neuronx==2.1.2.2.3.2 \
    transformers-neuronx==0.12.313 \
    libneuronxla==2.0.5347.0 \
    --extra-index-url=https://pip.repos.neuron.amazonaws.com

# Install HuggingFace packages
RUN pip3 install \
    hf_transfer huggingface_hub

# Install optimum-neuron
COPY --from=optimum-neuron /optimum-neuron optimum-neuron
RUN pip3 install ./optimum-neuron

# TGI base env
ENV HUGGINGFACE_HUB_CACHE=/tmp \
    HF_HUB_ENABLE_HF_TRANSFER=1 \
    PORT=80

# Disable color logs as they are not supported by CloudWatch
ENV LOGURU_COLORIZE=NO
ENV LOG_COLORIZE=0

# Install router
COPY --from=builder /usr/src/target/release/text-generation-router-v2 /usr/local/bin/text-generation-router
# Install launcher
COPY --from=builder /usr/src/target/release/text-generation-launcher /usr/local/bin/text-generation-launcher
# Install python server
COPY --from=pyserver /pyserver/build/dist dist
RUN pip install dist/text_generation_server*.tar.gz

# AWS Sagemaker compatible image
FROM neuron as sagemaker

COPY --from=optimum-neuron /optimum-neuron/text-generation-inference/sagemaker-entrypoint.sh entrypoint.sh
RUN chmod +x entrypoint.sh

ENTRYPOINT ["./entrypoint.sh"]


RUN apt-get update && apt-get install -y --no-install-recommends curl unzip \
    && rm -rf /var/lib/apt/lists/*

RUN HOME_DIR=/root && \
    pip install requests && \
    curl -o ${HOME_DIR}/oss_compliance.zip https://aws-dlinfra-utilities.s3.amazonaws.com/oss_compliance.zip && \
    unzip ${HOME_DIR}/oss_compliance.zip -d ${HOME_DIR}/ && \
    cp ${HOME_DIR}/oss_compliance/test/testOSSCompliance /usr/local/bin/testOSSCompliance && \
    chmod +x /usr/local/bin/testOSSCompliance && \
    chmod +x ${HOME_DIR}/oss_compliance/generate_oss_compliance.sh && \
    ${HOME_DIR}/oss_compliance/generate_oss_compliance.sh ${HOME_DIR} python && \
    rm -rf ${HOME_DIR}/oss_compliance*

RUN echo "N.B.: Although this image is released under the Apache-2.0 License, the Dockerfile used to build the image \
    has an indirect documentation dependency on third party <docutils/tools/editors/emacs/rst.el> project. The \
    <docutils/tools/editors/emacs/rst.el> project's licensing includes the <GPL v3> license. \
    \n\n\
    N.B.: Although this image is released under the Apache-2.0 License, the Dockerfile used to build the image uses the \
    third party <Text Generation Inference (TGI)> project. The <Text Generation Inference (TGI)> project's licensing \
    includes the <HFOIL --> https://github.com/huggingface/text-generation-inference/blob/main/LICENSE> \
    license." > /root/THIRD_PARTY_LICENSES

LABEL dlc_major_version="1"
LABEL com.amazonaws.ml.engines.sagemaker.dlc.framework.huggingface.tgi="true"
LABEL com.amazonaws.sagemaker.capabilities.accept-bind-to-port="true"
