# syntax=docker/dockerfile:1

############################
# Base Node Image
############################
FROM node:22-slim AS base

ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
ENV CI=true

RUN corepack enable


############################
# Build Go shared library
############################
FROM golang:1.24 AS go-build

WORKDIR /app

COPY apps/api/sharedLibs/go-html-to-md ./sharedLibs/go-html-to-md

RUN cd sharedLibs/go-html-to-md && \
    go mod download && \
    go build -o libhtml-to-markdown.so -buildmode=c-shared html-to-markdown.go


############################
# Build Stage
############################
FROM base AS build

WORKDIR /app

# System dependencies
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
    pkg-config \
    python3 \
    git \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Rust setup
ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --no-modify-path \
    && chmod -R a+w $RUSTUP_HOME $CARGO_HOME

############################
# Copy ONLY app source (correct monorepo path)
############################
COPY apps/api ./apps/api

WORKDIR /app/apps/api

# Safety check
RUN test -f package.json || (echo "package.json missing in apps/api" && exit 1)

############################
# Install dependencies
############################
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile


############################
# Build app
############################
RUN pnpm run build

RUN pnpm prune --prod --ignore-scripts


############################
# Runtime Stage
############################
FROM base AS runtime

WORKDIR /app

RUN apt-get update && apt-get install -y \
    git \
    procps \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8080

############################
# Copy build output
############################
COPY --from=build /app/apps/api/node_modules ./node_modules
COPY --from=build /app/apps/api/dist ./dist
COPY --from=build /app/apps/api/native ./native

# Copy Go library
COPY --from=go-build /app/sharedLibs/go-html-to-md/libhtml-to-markdown.so ./sharedLibs/go-html-to-md/

CMD ["node", "dist/src/harness.js", "--start-docker"]
