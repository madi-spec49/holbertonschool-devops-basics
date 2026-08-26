# syntax=docker/dockerfile:1
FROM golang:1.25-alpine AS build
WORKDIR /workspace

COPY go.mod ./
RUN go mod download

COPY cmd/ ./cmd/
COPY internal/ ./internal/

RUN go test ./...
RUN CGO_ENABLED=0 go build -buildvcs=false -o /out/inventory-api ./cmd/api

FROM scratch
COPY --from=build /out/inventory-api /usr/local/bin/inventory-api
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/inventory-api"]
