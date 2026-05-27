# syntax=docker/dockerfile:1

FROM golang:1.25-alpine AS builder

# Narzędzia do klonowania repo przez SSH + certyfikaty SSL dla połączeń HTTPS
RUN apk add --no-cache ca-certificates git openssh-client

WORKDIR /app

# Klonowanie kodu z GitHub przez SSH-agent (BuildKit --mount=type=ssh)
# Klucz SSH nie trafia do warstwy obrazu — mount istnieje tylko podczas wykonania RUN
RUN --mount=type=ssh \
    mkdir -p -m 0700 ~/.ssh && \
    ssh-keyscan github.com >> ~/.ssh/known_hosts && \
    git clone git@github.com:nazln/weather-app.git .

# Osobna warstwa dla zależności — cache nie jest unieważniany przy zmianie kodu
RUN go mod download

# Statyczny binarny plik — CGO_ENABLED=0 wymagane dla obrazu scratch
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o weather-app main.go

FROM scratch

LABEL org.opencontainers.image.authors="Nazarii Loboda" \
      org.opencontainers.image.title="Aplikacja Pogodowa" \
      org.opencontainers.image.description="Aplikacja webowa pokazująca aktualną pogodę" \
      org.opencontainers.image.version="1.0.0"

# Certyfikaty SSL potrzebne do połączeń HTTPS z API Open-Meteo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/weather-app /weather-app
COPY --from=builder /app/templates/ /templates/

EXPOSE 8089

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD ["/weather-app", "-health-check"]

ENTRYPOINT ["/weather-app"]
