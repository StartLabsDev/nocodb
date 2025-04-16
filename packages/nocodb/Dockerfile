###########
# Litestream Builder
###########
FROM golang:bullseye AS lt-builder

WORKDIR /usr/src/

RUN apt-get update && apt-get install -y \
    git \
    make \
    gcc \
    libc-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# build litestream
RUN git clone https://github.com/benbjohnson/litestream.git litestream
RUN cd litestream && go install ./cmd/litestream
RUN cp $GOPATH/bin/litestream /usr/src/lt

###########
# Builder
###########
FROM node:22-slim AS builder
WORKDIR /usr/src/app

# Install dependencies required for node-gyp and other build tools
RUN apt-get update && apt-get install -y \
    python3 \
    python3-distutils \
    python-is-python3 \
    make \
    g++ \
    libssl-dev \
    jq \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN python3 --version && python --version

# Install pnpm
RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

# Copy the entire monorepo structure
COPY --link ../../packages/ ./packages/
COPY --link ../../package.json ./package.json
COPY --link ../../pnpm-lock.yaml ./pnpm-lock.yaml
COPY --link ../../pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --link ../../.npmrc ./npmrc

# Configure pnpm to use hoisted node_modules for modclean compatibility
RUN echo "node-linker=hoisted" >> .npmrc

# Create a temporary package.json without the prepare script
RUN jq 'del(.scripts.prepare)' package.json > temp-package.json && mv temp-package.json package.json

# Remove postinstall scripts from packages that might cause issues
RUN jq 'del(.scripts.postinstall)' ./packages/nc-gui/package.json > temp-package.json && mv temp-package.json ./packages/nc-gui/package.json

# Install production dependencies, reduce node_modules size with modclean,
# remove sqlite deps, and add execute permission to start.sh
RUN pnpm install --prod --shamefully-hoist \
    && pnpm dlx modclean --patterns="default:*" --ignore="nc-lib-gui/**,dayjs/**,express-status-monitor/**,@azure/msal-node/dist/**,@react-email/**" --run \
    && rm -rf ./node_modules/sqlite3/deps \
    && chmod +x ./packages/nocodb/docker/start-litestream.sh

##########
# Runner
##########
FROM node:22-slim
WORKDIR /usr/src/app

ENV LITESTREAM_S3_SKIP_VERIFY=false \
    LITESTREAM_RETENTION=1440h \
    LITESTREAM_RETENTION_CHECK_INTERVAL=72h \
    LITESTREAM_SNAPSHOT_INTERVAL=24h \
    LITESTREAM_SYNC_INTERVAL=60s \
    NC_DOCKER=0.6 \
    NC_TOOL_DIR=/usr/app/data/ \
    NODE_ENV=production \
    PORT=8080

# Install additional runtime dependencies
RUN apt-get update && apt-get install -y \
    dumb-init \
    curl \
    wget \
    && curl -L "https://github.com/TomWright/dasel/releases/download/v2.8.1/dasel_linux_$(dpkg --print-architecture)" -o /usr/local/bin/dasel \
    && chmod +x /usr/local/bin/dasel \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy litestream binary and config file
COPY --link --from=lt-builder /usr/src/lt /usr/local/bin/litestream
COPY --link ./packages/nocodb/docker/litestream.yml /etc/litestream.yml

# Copy production code & main entry file
COPY --link --from=builder /usr/src/app/packages/nocodb/ /usr/src/app/
COPY --link --from=builder /usr/src/app/packages/nocodb/docker/start-litestream.sh /usr/src/appEntry/start.sh
RUN chmod +x /usr/src/appEntry/start.sh

EXPOSE 8080
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Start Nocodb
CMD ["/usr/src/appEntry/start.sh"]
