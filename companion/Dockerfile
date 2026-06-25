# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod ./
COPY . .
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -o companion .

FROM alpine:3.20
LABEL org.opencontainers.image.source="https://github.com/baseblocnh/LanPal"
RUN apk add --no-cache wget
COPY --from=builder /src/companion /companion
EXPOSE 5199
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s \
  CMD wget -qO- http://localhost:5199/status || exit 1
ENTRYPOINT ["/companion"]
