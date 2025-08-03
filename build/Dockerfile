# -----------------------------
# Global Build Arguments
# -----------------------------
ARG GO_VERSION=1.24
ARG WASM_API_VER=v2
ARG APP_VERSION=1.0.0

# -----------------------------
# Stage 1: Build Go backend + WASM
# -----------------------------
FROM golang:${GO_VERSION}-alpine AS build

ARG APP_VERSION
ARG WASM_API_VER

ENV CGO_ENABLED=0 \
    GO111MODULE=on

WORKDIR /tmp/playground

# Copy module files
COPY go.mod go.sum ./
RUN go mod download

# Copy sources
COPY cmd ./cmd
COPY pkg ./pkg
COPY internal ./internal
COPY tools ./tools

# Build Go backend and WASM binary
RUN echo "Building server with version ${APP_VERSION}" && \
    go build -o server -ldflags="-X 'main.Version=${APP_VERSION}'" ./cmd/playground && \
    GOOS=js GOARCH=wasm go build \
    -buildvcs=false \
    -ldflags="-s -w" \
    -trimpath \
    -o ./analyzer@${WASM_API_VER}.wasm ./cmd/wasm/analyzer && \
    go run ./tools/pkgindexer imports -o ./data/imports.json && \
    go run ./tools/pkgindexer index -o ./data/go-index.json

# -----------------------------
# Stage 2: Runtime container (no frontend)
# -----------------------------
FROM golang:${GO_VERSION}-alpine AS production

ENV GOROOT=/usr/local/go \
    APP_CLEAN_INTERVAL=10m \
    APP_DEBUG=false \
    APP_PLAYGROUND_URL=https://go.dev/_ \
    APP_GTAG_ID=

WORKDIR /opt/playground

# Copy built server and WASM assets
COPY --from=build /tmp/playground/server .
COPY --from=build /tmp/playground/*.wasm ./public/wasm/
COPY --from=build /tmp/playground/*.js ./public/wasm/
COPY --from=build /tmp/playground/data ./public/data

# Optional: if no frontend at all, ensure `./public` exists
RUN mkdir -p ./public

EXPOSE 8000

ENTRYPOINT ["/opt/playground/server", "-addr=:8000"]
