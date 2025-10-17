FROM rust:1.84-bookworm AS builder

ARG commitHash
ENV MEMPOOL_COMMIT_HASH=${commitHash}

WORKDIR /build

RUN apt-get update && \
    apt-get install -y curl ca-certificates && \
    curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs=22.14.0-1nodesource1 build-essential python3 pkg-config && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

COPY . .

ENV PATH="/usr/local/cargo/bin:$PATH"

# Build rust-gbt
WORKDIR /build/rust/gbt
RUN npm run build-release

# Build backend
WORKDIR /build/backend

# Configure npm for better network handling
RUN npm config set fetch-timeout 300000 && \
    npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm config set registry https://registry.npmjs.org/

# Copy rust-gbt to backend
RUN mkdir -p rust-gbt && \
    cp -r ../rust/gbt/pkg/* rust-gbt/

# Use npm ci for faster, more reliable installs with retries
RUN npm ci --omit=dev --omit=optional || \
    (npm cache clean --force && npm ci --omit=dev --omit=optional) || \
    (sleep 10 && npm ci --omit=dev --omit=optional)

RUN npm run build

# Package the application
RUN npm run package

FROM rust:1.84-bookworm AS runtime

RUN apt-get update && \
    apt-get install -y curl ca-certificates && \
    curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs=22.14.0-1nodesource1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /backend

RUN chown 1000:1000 ./
COPY --from=builder --chown=1000:1000 /build/backend/package ./package/
COPY --from=builder --chown=1000:1000 /build/backend/GeoIP ./GeoIP/ 2>/dev/null || true
COPY --from=builder --chown=1000:1000 /build/docker/backend/mempool-config.json ./
COPY --from=builder --chown=1000:1000 /build/docker/backend/start.sh ./
COPY --from=builder --chown=1000:1000 /build/docker/backend/wait-for-it.sh ./

USER 1000

EXPOSE 8999

CMD ["/backend/start.sh"]
