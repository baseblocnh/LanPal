# syntax=docker/dockerfile:1
# Build from the REPO ROOT, not from app/:
#   docker buildx build --platform linux/amd64,linux/arm64 \
#     -f app/Dockerfile -t ghcr.io/baseblocnh/lanpal:latest --push .

FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS go-builder
WORKDIR /src
COPY companion/go.mod ./
COPY companion/ .
RUN go mod tidy
RUN CGO_ENABLED=0 GOOS=linux   GOARCH=amd64       go build -ldflags="-s -w" -o companion-linux-amd64    . && \
    CGO_ENABLED=0 GOOS=linux   GOARCH=arm64       go build -ldflags="-s -w" -o companion-linux-arm64    . && \
    CGO_ENABLED=0 GOOS=linux   GOARCH=arm GOARM=7 go build -ldflags="-s -w" -o companion-linux-armv7    . && \
    CGO_ENABLED=0 GOOS=linux   GOARCH=arm GOARM=5 go build -ldflags="-s -w" -o companion-linux-armv5    . && \
    CGO_ENABLED=0 GOOS=windows GOARCH=amd64       go build -ldflags="-s -w" -o companion-windows-amd64.exe . && \
    CGO_ENABLED=0 GOOS=darwin  GOARCH=amd64       go build -ldflags="-s -w" -o companion-darwin-amd64   . && \
    CGO_ENABLED=0 GOOS=darwin  GOARCH=arm64       go build -ldflags="-s -w" -o companion-darwin-arm64   .

FROM python:3.12-slim
LABEL org.opencontainers.image.source="https://github.com/baseblocnh/LanPal"
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*

COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .
RUN mkdir -p static/downloads
COPY --from=go-builder /src/companion-* static/downloads/

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD curl -f http://localhost:8080/api/servers || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
