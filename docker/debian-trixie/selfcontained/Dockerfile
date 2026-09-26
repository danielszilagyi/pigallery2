#-----------------BUILDER-----------------
#-----------------------------------------
FROM node:22-trixie AS nodebase

FROM debian:forky-slim AS builder

COPY --from=nodebase /usr/local/ /usr/local/

ENV SHARP_FORCE_GLOBAL_LIBVIPS=1

# inspired by: https://dev.to/ilyasa1211/converting-heic-image-extension-in-nodejs-with-the-sharp-library-39mg
# also see: https://github.com/bpatrik/pigallery2/issues/965

# Install build and vips dependencies
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl python3 git build-essential pkg-config \
  libvips-dev \
  libheif-plugin-libde265 && \
  rm -rf /var/lib/apt/lists/*

# copying only package{-lock}.json to make node_modules cachable
RUN git clone -b 3_5_2 https://github.com/danielszilagyi/pigallery2.git /build
WORKDIR /build
RUN npm install --unsafe-perm \
    && mkdir -p /build/release/data/config \
    && mkdir -p /build/release/data/db \
    && mkdir -p /build/release/data/images \
    && mkdir -p /build/release/data/tmp \
    && npm run create-release -- \
         --skip-opt-packages=ffmpeg-static,ffprobe-static \
         --force-opt-packages
WORKDIR /build/release
RUN npm install --unsafe-perm --no-package-lock node-addon-api@8.5.0 node-gyp@11.5.0 \
    && npm explore sharp -- npm run build \
    && npm install \
    --package-lock-only \
    --ignore-scripts \
    && npm audit --omit=dev || true \
    && rm -rf \
      /build/release/node_modules/@img/sharp-linux-* \
      /build/release/node_modules/@img/sharp-linuxmusl-* \
      /build/release/node_modules/@img/sharp-libvips-*


#-----------------MAIN--------------------
#-----------------------------------------
FROM debian:forky-slim AS main

WORKDIR /app

ENV NODE_ENV=production \
    default-Database-dbFolder=/app/data/db \
    default-Media-folder=/app/data/images \
    default-Media-tempFolder=/app/data/tmp \
    default-Extensions-folder=/app/data/config/extensions \
    PI_DOCKER=true

ARG TARGETARCH
ARG DEBIAN_MIRROR=http://repo.jztkft.hu/debian
ARG DEBIAN_SECURITY_MIRROR=http://deb.debian.org/debian-security

RUN sed -i \
      -e "s|^URIs: http://deb.debian.org/debian$|URIs: ${DEBIAN_MIRROR}|" \
      -e "s|^URIs: http://deb.debian.org/debian-security$|URIs: ${DEBIAN_SECURITY_MIRROR}|" \
      /etc/apt/sources.list.d/debian.sources \
    && cat /etc/apt/sources.list.d/debian.sources \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
         ca-certificates \
         wget \
         ffmpeg \
         libvips42t64 \
         libheif-plugin-libde265 \
    && if [ "$TARGETARCH" = "amd64" ]; then \
         apt-get install -y --no-install-recommends intel-media-va-driver; \
       fi \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /usr/local/bin/node /usr/local/bin/node

COPY --from=builder /build/release /app

# debug
RUN node -e 'console.log(require("sharp").versions)'

RUN ["node","--expose-gc","./src/backend/index","--run-diagnostics","--config-path=/app/diagnostics-config.json","--Server-Log-level=silly"]

EXPOSE 80

HEALTHCHECK --interval=40s --timeout=30s --retries=3 --start-period=60s \
  CMD wget --quiet --tries=1 --no-check-certificate --spider \
      http://127.0.0.1:80/heartbeat || exit 1

ENTRYPOINT ["node","--expose-gc","./src/backend/index","--config-path=/app/data/config/config.json"]
