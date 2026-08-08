# syntax=docker/dockerfile:1
#
# Static Astro build (output: 'static') → the shared nginx runtime from this
# template. Copy to the root of your project as `Dockerfile`.
#
# The server config is NOT baked in: it is mounted from the app-nginx-config
# ConfigMap that this template's base/ generates. To run the image locally the
# way the cluster does, mount it yourself:
#
#   docker build -t mysite:local .
#   docker run --rm -p 8080:8080 \
#     -v "$PWD/nginx-default.conf:/etc/nginx/conf.d/default.conf:ro" \
#     mysite:local

# Pin to a nginx-sha-<commit> tag for reproducible builds.
ARG NGINX_IMAGE=ghcr.io/nickmous/astro-k8s-deployment-kit:nginx-latest

FROM node:24-trixie-slim AS build
WORKDIR /app

# Astro bakes the site URL into the output (canonical links, sitemap, RSS), so
# it has to be known here rather than at deploy time. Both deploy workflows
# pass it: the production URL, or https://<branch-slug>.staging.example.com.
# Read it in astro.config.mjs:
#
#   site: process.env.SITE_URL || 'https://example.com',
ARG SITE_URL=""
ARG SENTRY_RELEASE=""
ENV SITE_URL=${SITE_URL} \
    SENTRY_RELEASE=${SENTRY_RELEASE}

# Dependencies first, so a source-only change reuses this layer.
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Pre-compress text assets. nginx then serves the .br / .gz sibling directly
# via brotli_static / gzip_static — best ratio, zero per-request CPU.
RUN apt-get update \
    && apt-get install -y --no-install-recommends brotli \
    && rm -rf /var/lib/apt/lists/* \
    && find dist -type f \( \
           -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \
           -o -name '*.svg' -o -name '*.json' -o -name '*.xml' -o -name '*.txt' \
       \) -size +256c \
       -exec gzip -9 -k -f {} + \
       -exec brotli -q 11 -k -f {} +

FROM ${NGINX_IMAGE} AS runtime

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 8080
