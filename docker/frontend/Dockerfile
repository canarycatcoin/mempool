FROM node:22.14.0-bookworm-slim AS builder

ARG commitHash
ENV DOCKER_COMMIT_HASH=${commitHash}
ENV CYPRESS_INSTALL_BINARY=0

WORKDIR /build
COPY . .

RUN apt-get update && \
    apt-get install -y build-essential rsync && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Configure npm for better network handling
RUN npm config set fetch-timeout 300000 && \
    npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm config set registry https://registry.npmjs.org/

WORKDIR /build/frontend

# Copy sample config if needed
RUN if [ -f mempool-frontend-config.sample.json ]; then \
      cp mempool-frontend-config.sample.json mempool-frontend-config.json; \
    fi

# Use npm ci for faster, more reliable installs with retries
RUN npm ci --omit=dev --omit=optional || \
    (npm cache clean --force && npm ci --omit=dev --omit=optional) || \
    (sleep 10 && npm ci --omit=dev --omit=optional)

# Unset MEMPOOL_CDN to prevent attempting to download assets from mempool.space
ENV MEMPOOL_CDN=
# Create necessary directory structure to prevent download errors
RUN mkdir -p src/resources/mining-pools && \
    touch src/resources/mining-pools/.gitkeep

# Build the application (sync-assets will skip downloads with MEMPOOL_CDN unset)
RUN npm run build

FROM nginx:1.27.0-alpine

WORKDIR /patch

COPY --from=builder /build/docker/frontend/entrypoint.sh .
COPY --from=builder /build/docker/frontend/wait-for .
COPY --from=builder /build/frontend/dist/mempool /var/www/mempool
COPY --from=builder /build/nginx.conf /etc/nginx/
COPY --from=builder /build/nginx-mempool.conf /etc/nginx/conf.d/

RUN chmod +x /patch/entrypoint.sh && \
    chmod +x /patch/wait-for

RUN chown -R 1000:1000 /patch && chmod -R 755 /patch && \
        chown -R 1000:1000 /var/cache/nginx && \
        chown -R 1000:1000 /var/log/nginx && \
        chown -R 1000:1000 /etc/nginx/nginx.conf && \
        chown -R 1000:1000 /etc/nginx/conf.d && \
        chown -R 1000:1000 /var/www/mempool

RUN touch /var/run/nginx.pid && \
        chown -R 1000:1000 /var/run/nginx.pid

USER 1000

ENTRYPOINT ["/patch/entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
