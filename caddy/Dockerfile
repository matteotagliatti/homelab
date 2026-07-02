# Fully qualified names — on Fedora/Bazzite Podman resolves short names to
# registry.fedoraproject.org, which has no Caddy image.
#
# Pinned to Caddy 2.9.1: github.com/libdns/vercel is unmaintained and still
# uses the pre-1.0 libdns API, which breaks builds on Caddy 2.10+.
FROM docker.io/library/caddy:2.9.1-builder AS builder

RUN xcaddy build v2.9.1 \
	--with github.com/caddy-dns/vercel

FROM docker.io/library/caddy:2.9.1-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
