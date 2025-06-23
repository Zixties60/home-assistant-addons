ARG NQPTP_BRANCH=main
ARG SHAIRPORT_SYNC_BRANCH=master
ARG BUILD_FROM=ghcr.io/home-assistant/aarch64-base:latest
FROM alpine:3.20 AS builder

RUN apk -U add \
        alsa-lib-dev \
        autoconf \
        automake \
        avahi-dev \
        build-base \
        dbus \
        ffmpeg-dev \
        git \
        libconfig-dev \
        libgcrypt-dev \
        libplist-dev \
        libressl-dev \
        libsndfile-dev \
        libsodium-dev \
        libtool \
        pipewire-dev \
        mosquitto-dev \
        popt-dev \
        pulseaudio-dev \
        soxr-dev \
        xxd

##### ALAC #####
FROM builder AS alac
RUN git clone --depth=1 https://github.com/mikebrady/alac
WORKDIR /alac
RUN autoreconf -i
RUN ./configure
RUN make -j $(nproc)
RUN make install
WORKDIR /
##### ALAC END #####

##### NQPTP #####
FROM builder AS nqptp
ARG NQPTP_BRANCH
RUN git clone --depth=1 -b "$NQPTP_BRANCH" https://github.com/mikebrady/nqptp
WORKDIR /nqptp
RUN autoreconf -i
RUN ./configure
RUN make -j $(nproc)
WORKDIR /
##### NQPTP END #####

##### SPS #####
# Note: apple-alac requires alac build first.
FROM alac AS shairport-sync
ARG SHAIRPORT_SYNC_BRANCH
WORKDIR /
RUN git clone -b "$SHAIRPORT_SYNC_BRANCH" https://github.com/mikebrady/shairport-sync
WORKDIR /shairport-sync/build
RUN autoreconf -i ../
RUN CFLAGS="-O3" CXXFLAGS="-O3" ../configure --sysconfdir=/etc --with-alsa --with-pa --with-soxr --with-avahi --with-ssl=openssl \
        --with-airplay-2 --with-metadata --with-dummy --with-pipe --with-dbus-interface \
        --with-stdout --with-mpris-interface --with-mqtt-client \
        --with-apple-alac --with-convolution --with-pw
RUN make -j $(nproc)
RUN DESTDIR=install make install
WORKDIR /
##### SPS END #####

# ##### STATIC FILES #####
# FROM scratch AS files

# # # Add run script that will start SPS
# COPY ./docker/run.sh ./run.sh
# COPY ./docker/etc/s6-overlay/s6-rc.d /etc/s6-overlay/s6-rc.d
# COPY ./docker/etc/pulse /etc/pulse
# ##### END STATIC FILES #####

##### BUILD FILES #####
FROM scratch AS build-files
COPY --from=shairport-sync /shairport-sync/build/install/usr/local/bin/shairport-sync /usr/local/bin/shairport-sync
COPY --from=shairport-sync /shairport-sync/build/install/usr/local/share/man/man1 /usr/share/man/man1
COPY --from=nqptp /nqptp/nqptp /usr/local/bin/nqptp
COPY --from=alac /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=shairport-sync /shairport-sync/build/install/etc/shairport-sync.conf /etc/
COPY --from=shairport-sync /shairport-sync/build/install/etc/shairport-sync.conf.sample /etc/
COPY --from=shairport-sync /shairport-sync/build/install/etc/dbus-1/system.d/shairport-sync-dbus.conf /etc/dbus-1/system.d/
COPY --from=shairport-sync /shairport-sync/build/install/etc/dbus-1/system.d/shairport-sync-mpris.conf /etc/dbus-1/system.d/
##### END BUILD FILES #####

# Shairport Sync Addon Runtime System
FROM ${BUILD_FROM} AS hassio-addon

ENV S6_CMD_WAIT_FOR_SERVICES=1
ENV S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0

RUN apk -U add \
        alsa-lib \
        avahi \
        avahi-tools \
        dbus \
        ffmpeg \
        glib \
        less \
        less-doc \
        libconfig \
        libgcrypt \
        libplist \
        libpulse \
        libressl4.1-libcrypto \
        libsndfile \
        libsodium \
        libuuid \
        pipewire \
        man-pages \
        mandoc \
        mosquitto \
        popt \
        soxr \
        curl

RUN rm -rfv /lib/apk/db/* && \
    rm -rfv /etc/avahi/services/*.service && \
    addgroup shairport-sync && \
    adduser -D shairport-sync -G shairport-sync && \
    addgroup -g 29 docker_audio && \
    addgroup shairport-sync docker_audio && \
    addgroup shairport-sync audio && \
    mkdir -p /run/dbus

# Remove anything we don't need.
# Remove any statically-defined Avahi services, e.g. SSH and SFTP

# Create non-root user for running the container -- running as the user 'shairport-sync' also allows
# Shairport Sync to provide the D-Bus and MPRIS interfaces within the container
# Add the shairport-sync user to the pre-existing audio group, which has ID 29, for access to the ALSA stuff

# COPY --from=files / /
COPY --from=build-files / /

# Copy root filesystem
COPY rootfs /

# CHMODS
RUN chmod +x /run.sh
RUN chmod +x /etc/s6-overlay/s6-rc.d/01-startup/script.sh
RUN chmod +x /etc/s6-overlay/s6-rc.d/02-dbus/20-dbus.sh
RUN chmod +x /etc/s6-overlay/s6-rc.d/03-avahi/30-avahi.sh

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
