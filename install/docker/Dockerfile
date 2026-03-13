# =============================================================================
# QMD — Multi-stage Docker build
#
# Stage 1 (builder): installs all deps (including native) and compiles TypeScript.
# Stage 2 (runtime): minimal image with compiled output + Bun runtime.
# =============================================================================

# ---- Stage 1: builder -------------------------------------------------------
FROM node:22-bookworm-slim AS builder

# Build tools required by native modules:
#   better-sqlite3  → python3, make, g++, libsqlite3-dev
#   node-llama-cpp  → python3, make, g++, cmake
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 make g++ cmake libsqlite3-dev curl \
    && rm -rf /var/lib/apt/lists/*

# Install Bun (used to install deps and run the app)
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:$PATH"

WORKDIR /app

# Install dependencies first (layer-cached unless lockfile changes)
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

# Copy source and compile TypeScript → dist/
COPY . .
RUN npm run build


# ---- Stage 2: runtime -------------------------------------------------------
FROM node:22-bookworm-slim AS runtime

# Runtime libraries for SQLite native extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
        libsqlite3-0 curl \
    && rm -rf /var/lib/apt/lists/*

# Install Bun (the shell wrapper in bin/qmd uses bun.lock to auto-detect Bun)
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:$PATH"

WORKDIR /app

# Copy only what is needed to run the compiled app
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/bin          ./bin
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/bun.lock     ./bun.lock

# XDG dirs are remapped to /data/* so host volumes land in the right places
ENV XDG_CACHE_HOME=/data/cache
ENV XDG_CONFIG_HOME=/data/config
ENV HOME=/root

# Bind MCP HTTP server on all interfaces so peer containers can reach it
ENV QMD_HTTP_HOST=0.0.0.0

# Internal port (always 8181 inside the container; host mapping is in compose)
EXPOSE 8181

# Start the MCP server in HTTP mode (foreground, no daemon)
CMD ["bun", "dist/cli/qmd.js", "mcp", "--http", "--port", "8181"]
