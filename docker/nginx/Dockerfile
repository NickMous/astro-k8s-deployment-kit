# Shared nginx runtime image for static Astro sites.
#
# It is the unprivileged nginx image (non-root, uid 101, listens above 1024)
# plus the ngx_brotli dynamic modules, so `brotli_static` can serve the .br
# files produced at build time at zero per-request CPU cost.
#
# Project Dockerfiles use this as their runtime stage:
#
#   FROM ghcr.io/nickmous/astro-k8s-deployment-kit:nginx-latest AS runtime
#   COPY --from=build /app/dist /usr/share/nginx/html
#
# The server config is NOT baked in — it is mounted from the ConfigMap in
# this repo's base/ kustomization, so config changes ship via a ?ref= bump
# instead of an image rebuild.
#
# Dynamic modules must be compiled against the exact nginx version they load
# into, so the builder stage starts from the same image and reads its version.

ARG NGINX_IMAGE_TAG=1.30-alpine

FROM nginxinc/nginx-unprivileged:${NGINX_IMAGE_TAG} AS builder

USER root

RUN apk add --no-cache \
        build-base \
        pcre-dev \
        zlib-dev \
        openssl-dev \
        linux-headers \
        cmake \
        git \
        curl

WORKDIR /build

RUN git clone --depth=1 --recurse-submodules --shallow-submodules \
        https://github.com/google/ngx_brotli.git

RUN cd ngx_brotli/deps/brotli \
    && mkdir -p out && cd out \
    && cmake -DCMAKE_BUILD_TYPE=Release \
             -DBUILD_SHARED_LIBS=OFF \
             -DCMAKE_C_FLAGS="-O2" \
             -DCMAKE_CXX_FLAGS="-O2" \
             -DCMAKE_INSTALL_PREFIX=./installed \
             .. \
    && cmake --build . --config Release --target brotlienc

RUN NGINX_VERSION=$(nginx -v 2>&1 | sed -n 's/^nginx version: nginx\/\(.*\)$/\1/p') \
    && echo "Building modules for nginx ${NGINX_VERSION}" \
    && curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xz \
    && cd "nginx-${NGINX_VERSION}" \
    && ./configure --with-compat --add-dynamic-module=/build/ngx_brotli \
    && make -j"$(nproc)" modules \
    && install -m 644 objs/ngx_http_brotli_filter_module.so /tmp/ \
    && install -m 644 objs/ngx_http_brotli_static_module.so /tmp/

FROM nginxinc/nginx-unprivileged:${NGINX_IMAGE_TAG}

USER root

COPY --from=builder /tmp/ngx_http_brotli_filter_module.so /etc/nginx/modules/
COPY --from=builder /tmp/ngx_http_brotli_static_module.so /etc/nginx/modules/

# load_module only works in the main context (before events{}), so prepend it
# to the image's nginx.conf rather than dropping a file into conf.d/.
RUN printf 'load_module modules/ngx_http_brotli_filter_module.so;\nload_module modules/ngx_http_brotli_static_module.so;\n' \
        | cat - /etc/nginx/nginx.conf > /tmp/nginx.conf \
    && mv /tmp/nginx.conf /etc/nginx/nginx.conf \
    && chmod 644 /etc/nginx/nginx.conf \
    && nginx -t

# Back to the unprivileged image's default user.
USER 101

EXPOSE 8080
