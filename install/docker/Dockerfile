# =============================================================================
# QMD — Multi-stage Docker build
#
# Stage 1 (builder): installs deps and compiles TypeScript → dist/
# Stage 2 (runtime): minimal Node.js image with compiled output only
# =============================================================================

# ---- Stage 1: builder -------------------------------------------------------
FROM node:22-bookworm-slim AS builder

# Build tools required by native modules:
#   better-sqlite3  → python3, make, g++, libsqlite3-dev
#   node-llama-cpp  → python3, make, g++, cmake
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 make g++ cmake libsqlite3-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install dependencies first (layer-cached unless package.json changes)
COPY package.json ./
RUN npm install

# Copy source and compile TypeScript → dist/
COPY . .
RUN npm run build


# ---- Stage 2: runtime -------------------------------------------------------
FROM node:22-bookworm-slim AS runtime

# Runtime libraries for native SQLite extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
        libsqlite3-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only what is needed to run the compiled app
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

# XDG dirs are remapped to /data/* so host volumes land in the right places
ENV XDG_CACHE_HOME=/data/cache
ENV XDG_CONFIG_HOME=/data/config
ENV HOME=/root

# Bind MCP HTTP server on all interfaces so peer containers can reach it
ENV QMD_HTTP_HOST=0.0.0.0

# Internal port (always 8181 inside the container; host mapping is in compose)
EXPOSE 8181

# Start the MCP server in HTTP mode (foreground, no daemon)
CMD ["node", "dist/cli/qmd.js", "mcp", "--http", "--port", "8181"]
