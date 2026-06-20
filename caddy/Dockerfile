# syntax=docker/dockerfile:1.7
# ============================================================================
# Custom Caddy image with the Cloudflare DNS plugin.
#
# Why: the merchant frontend serves *.storebaze.com (wildcard subdomain). A
# wildcard TLS cert from Let's Encrypt requires the DNS-01 challenge, which
# needs a DNS-provider plugin. We use Cloudflare here.
# ============================================================================

FROM caddy:2.8-builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:2.8-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
