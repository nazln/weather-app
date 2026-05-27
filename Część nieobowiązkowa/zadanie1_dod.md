# Aplikacja Pogodowa — PAwChO Zadanie 1 — Część Dodatkowa
## Wymagania wstępne — skanowanie CVE

Przed realizacją części dodatkowej przeprowadzono analizę podatności obrazu za pomocą narzędzia **Docker Scout**.

Podczas pierwszego skanowania wykryto 9 podatności HIGH oraz 7 MEDIUM w pakiecie `stdlib 1.24.13`. Wszystkie podatności zostały naprawione przez aktualizację obrazu bazowego z `golang:1.24-alpine` do `golang:1.25-alpine` oraz aktualizację `go.mod` do wersji Go 1.25.

Po aktualizacji przeprowadzono ponowne skanowanie:

```
$ docker scout cves nazariil/weather-app:v1.0

✓ No vulnerable package detected

                   │               Analyzed Image
───────────────────┼────────────────────────────────────────────
 Target            │  nazariil/weather-app:v1.0
   digest          │  6501ff820c73
   platform        │ linux/amd64
   provenance      │ https://github.com/nazln/weather-app.git
                   │  9c05b023b12ca092bb30f5efdf413ca63d4b2a85
   vulnerabilities │    0C     0H     0M     0L
   size            │ 3.7 MB
   packages        │ 2

No vulnerable packages detected
```

Obraz nie zawiera żadnych podatności sklasyfikowanych jako CRITICAL lub HIGH.

---

## Rozszerzony frontend BuildKit — klonowanie kodu z GitHub przez SSH

W pliku `Dockerfile` zastosowano rozszerzony frontend BuildKit (`# syntax=docker/dockerfile:1`) oraz instrukcję `--mount=type=ssh`, która umożliwia klonowanie kodu bezpośrednio z publicznego repozytorium na GitHub podczas procesu budowania obrazu.

Klucz SSH nie trafia do żadnej warstwy obrazu — mount istnieje wyłącznie podczas wykonania instrukcji `RUN`.

Fragment pliku `Dockerfile`:
```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.25-alpine AS builder

RUN apk add --no-cache ca-certificates git openssh-client

WORKDIR /app

RUN --mount=type=ssh \
    mkdir -p -m 0700 ~/.ssh && \
    ssh-keyscan github.com >> ~/.ssh/known_hosts && \
    git clone git@github.com:nazln/weather-app.git .
```

Budowanie obrazu z wykorzystaniem SSH-agenta:
```bash
docker buildx build \
  --builder weather-builder \
  --platform linux/amd64,linux/arm64 \
  --ssh default \
  --cache-to type=registry,ref=nazariil/weather-app:cache,mode=max \
  --cache-from type=registry,ref=nazariil/weather-app:cache \
  --tag nazariil/weather-app:v1.0 \
  --push \
  .
```

---

## Budowa obrazu wieloplatformowego (linux/amd64, linux/arm64)

Obraz zbudowano przy użyciu buildera `weather-builder` opartego na sterowniku `docker-container`:

```bash
docker buildx create --name weather-builder --driver docker-container --bootstrap
docker buildx use weather-builder
```

Potwierdzenie obecności obu platform w manifeście:

```
$ docker buildx imagetools inspect nazariil/weather-app:v1.0

Name:      docker.io/nazariil/weather-app:v1.0
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:3987eeada110f0092c9c54a98276cdd8dd63bfbcc5ace20e46a51fe365b2b7ff

Manifests:
  Name:      docker.io/nazariil/weather-app:v1.0@sha256:090143f775d9440ffbcb5358c0d3deeaaf5206734a8f35d13c457bd1fa56ffae
  MediaType: application/vnd.oci.image.manifest.v1+json
  Platform:  linux/amd64

  Name:      docker.io/nazariil/weather-app:v1.0@sha256:2d71b5c5ed89a7ce496f13318341d520e8bdd1c535af9fd46eea4057a43df1e1
  MediaType: application/vnd.oci.image.manifest.v1+json
  Platform:  linux/arm64
```

Manifest zawiera deklaracje dla platform `linux/amd64` oraz `linux/arm64`.

---

## Dane cache (backend registry, tryb max)

Dane cache eksportowane są do osobnego obrazu w rejestrze DockerHub (`nazariil/weather-app:cache`) z wykorzystaniem backendu `registry` w trybie `max`. Tryb `max` buforuje wszystkie warstwy, włącznie z etapem builder, co znacząco skraca czas kolejnych kompilacji.

```bash
--cache-to type=registry,ref=nazariil/weather-app:cache,mode=max
--cache-from type=registry,ref=nazariil/weather-app:cache
```

Pierwsze uruchomienie — brak danych cache (oczekiwany błąd importu):
```
=> ERROR importing cache manifest from nazariil/weather-app:cache
```

Drugie uruchomienie — dane cache pobrane i wykorzystane (kilka sekund zamiast ~8 minut):
```
=> [linux/amd64 builder 2/6] RUN apk add ...    CACHED
=> [linux/amd64 builder 3/6] RUN --mount=type=ssh git clone ...    CACHED
=> [linux/amd64 builder 4/6] RUN go mod download    CACHED
=> [linux/amd64 builder 5/6] RUN go build ...    CACHED
=> [linux/arm64 builder 2/6] RUN apk add ...    CACHED
=> [linux/arm64 builder 3/6] RUN --mount=type=ssh git clone ...    CACHED
=> [linux/arm64 builder 4/6] RUN go mod download    CACHED
=> [linux/arm64 builder 5/6] RUN go build ...    CACHED
```

Obraz z danymi cache dostępny w rejestrze DockerHub jako `nazariil/weather-app:cache`.
