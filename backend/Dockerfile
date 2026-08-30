FROM golang:alpine3.24 AS compile

WORKDIR /compile

COPY go.mod go.sum /compile/

RUN go mod download

COPY . /compile/

RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o ./app.bin ./cmd/api/

FROM alpine:3.24 AS final 

RUN adduser -DH -g name svc-backend

USER svc-backend

WORKDIR /app

COPY --from=compile --chown=svc-backend:svc-backend /compile/app.bin /app/ 

EXPOSE 8081

ENTRYPOINT [ "./app.bin" ]

HEALTHCHECK --interval=20s --timeout=2s --start-period=10s --retries=3 CMD wget -q -O /dev/null http://localhost:8081/health || exit 1
