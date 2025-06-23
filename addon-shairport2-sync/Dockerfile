ARG BUILD_FROM=ghcr.io/home-assistant/armv7-base-debian:latest
FROM ${BUILD_FROM} AS builder

ENV LANG C.UTF-8

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt update && \
        apt install -y --no-install-recommends build-essential \
        autoconf \
        automake \
        avahi-daemon \
        git \
        iproute2 \
        libasound2-dev \
        libavcodec-dev \
        libavformat-dev \
        libavahi-client-dev \
        libavutil-dev \
        libconfig-dev \
        libgcrypt-dev \
        libplist-dev \
        libpopt-dev \
        libsodium-dev \
        libsoxr-dev \
        libssl-dev \
        libtool \
        uuid-dev \
        xxd \
        && rm -rf /var/lib/apt/lists/*

WORKDIR /root
RUN git clone https://github.com/mikebrady/shairport-sync.git 
WORKDIR /root/shairport-sync
RUN autoreconf -fi 
RUN ./configure \
        --with-alsa \
        --with-pipe \
        --with-avahi \
        --with-ssl=openssl \
        --with-soxr \
        --with-metadata \
        --with-airplay-2 
RUN make 
RUN make install

# Copy root filesystem
COPY rootfs /

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="Shairport Sync" \
    io.hass.description="Shairport Sync for Hass.io" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Zixties60"
