# syntax=docker/dockerfile:1.7

ARG UPSTREAM_IMAGE=nginx:1.30-alpine3.23
ARG ALPINE_IMAGE=alpine:3.23

FROM ${UPSTREAM_IMAGE} AS upstream
RUN nginx -V 2>&1 \
    | sed -n 's/^configure arguments: //p' \
    | sed -E 's#--add-dynamic-module=[^ ]+##g' \
    > /tmp/configure_args

FROM ${ALPINE_IMAGE} AS builder
RUN apk add --no-cache \
      gcc libc-dev make linux-headers findutils bash \
      openssl-dev pcre2-dev zlib-dev \
      libxslt-dev gd-dev geoip-dev
WORKDIR /usr/src/nginx
COPY . .
COPY --from=upstream /tmp/configure_args /tmp/configure_args
RUN addgroup -S -g 101 nginx \
 && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
 && eval "./auto/configure $(cat /tmp/configure_args) \
      --with-http_xslt_module=dynamic \
      --with-http_image_filter_module=dynamic \
      --with-http_geoip_module=dynamic \
      --with-stream_geoip_module=dynamic" \
 && make -j"$(nproc)" \
 && make DESTDIR=/stage install \
 && strip /stage/usr/sbin/nginx \
 && find /stage -name '*.so' -exec strip {} +

FROM ${ALPINE_IMAGE}
RUN apk add --no-cache \
      pcre2 openssl zlib libxslt libgd geoip \
      ca-certificates tzdata curl tini gettext-envsubst \
 && addgroup -S -g 101 nginx \
 && adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
 && mkdir -p /var/cache/nginx /var/log/nginx /etc/nginx/conf.d /docker-entrypoint.d

COPY --from=builder /stage/usr/sbin/nginx         /usr/sbin/nginx
COPY --from=builder /stage/usr/lib/nginx/modules/ /usr/lib/nginx/modules/

COPY --from=upstream /docker-entrypoint.sh /docker-entrypoint.sh
COPY --from=upstream /docker-entrypoint.d/ /docker-entrypoint.d/
COPY --from=upstream /etc/nginx/           /etc/nginx/
COPY --from=upstream /usr/share/nginx/     /usr/share/nginx/

RUN ln -sf /dev/stdout /var/log/nginx/access.log \
 && ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
