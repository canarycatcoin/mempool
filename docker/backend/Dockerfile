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

# Configure npm for better network handling
RUN npm config set fetch-timeout 300000 && \
    npm config set fetch-retry-mintimeout 20000 && \
    npm config set fetch-retry-maxtimeout 120000 && \
    npm config set fetch-retries 5 && \
    npm config set registry https://registry.npmjs.org/

# Build rust-gbt
WORKDIR /build/rust/gbt
# Install @napi-rs/cli first (needed by build script)
RUN npm install --no-save @napi-rs/cli@2.18.0
# Build rust-gbt
RUN npm run build -- --release --strip

# Copy rust-gbt output to backend directory
RUN mkdir -p /build/backend/rust-gbt && \
    cp index.js index.d.ts package.json *.node /build/backend/rust-gbt/ 2>/dev/null || true

# Build backend
WORKDIR /build/backend

# Install backend dependencies (with rust-gbt already built and copied)
RUN npm ci --omit=dev --omit=optional || \
    (npm cache clean --force && npm ci --omit=dev --omit=optional) || \
    (sleep 10 && npm ci --omit=dev --omit=optional)

# Build the backend
RUN npm run build

# Package the application
RUN npm run package

# Ensure GeoIP directory exists (create empty one if it doesn't)
RUN mkdir -p /build/backend/GeoIP

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
COPY --from=builder --chown=1000:1000 /build/backend/GeoIP ./GeoIP/
COPY --from=builder --chown=1000:1000 /build/docker/backend/mempool-config.json ./
COPY --from=builder --chown=1000:1000 /build/docker/backend/start.sh ./
COPY --from=builder --chown=1000:1000 /build/docker/backend/wait-for-it.sh ./

USER 1000

EXPOSE 8999

CMD ["/backend/start.sh"]
