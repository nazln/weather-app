# Aplikacja Pogodowa — PAwChO Zadanie 1
## Opis aplikacji

Aplikacja webowa napisana w Go, która wyświetla aktualną pogodę dla wybranego miasta.
Dane pobierane są z bezpłatnego API [Open-Meteo](https://open-meteo.com/) — bez klucza API.

Po uruchomieniu kontenera aplikacja zapisuje w logach:
- datę i godzinę uruchomienia
- imię i nazwisko autora
- numer portu TCP, na którym nasłuchuje

Użytkownik może wybrać miasto z predefiniowanej listy i sprawdzić temperaturę, prędkość wiatru oraz stan nieba.

Kod źródłowy: [`Część obowiązkowa/main.go`](Część%20obowiązkowa/main.go)  
Szablon HTML: [`Część obowiązkowa/templates/index.html`](Część%20obowiązkowa/templates/index.html)

---

## Plik Dockerfile

Pełna treść: [`Część obowiązkowa/Dockerfile`](Część%20obowiązkowa/Dockerfile)

Zastosowane rozwiązania:
- **Wieloetapowe budowanie** (`golang:1.25-alpine` - `scratch`) — obraz końcowy zawiera tylko binarny plik aplikacji
- **Obraz bazowy scratch** — minimalny rozmiar, brak zbędnych narzędzi systemowych
- **Rozszerzony frontend BuildKit** (`# syntax=docker/dockerfile:1`) — umożliwia użycie `--mount=type=ssh`
- **Klonowanie kodu z GitHub przez SSH** (`--mount=type=ssh`) — klucz SSH nie trafia do warstwy obrazu
- **Optymalizacja cache** — `go mod download` jako osobna warstwa przed kompilacją
- **HEALTHCHECK** — Docker sprawdza działanie aplikacji co 30 sekund
- **Metadane OCI** — etykieta `org.opencontainers.image.authors`

---

## Polecenia

### a. Budowanie obrazu

```bash
docker build -t nazariil/weather-app:v1.0 "Część obowiązkowa/"
```

### b. Uruchomienie kontenera

```bash
docker run -d -p 8089:8089 --name weather-app nazariil/weather-app:v1.0
```

Aplikacja dostępna pod adresem: http://localhost:8089

### c. Odczyt logów z uruchomienia kontenera

```bash
docker logs weather-app
```

Przykładowy wynik:
```
2025/05/27 12:00:00 === Aplikacja pogodowa ===
2025/05/27 12:00:00 Data uruchomienia: 2025-05-27 12:00:00
2025/05/27 12:00:00 Autor: Nazarii Loboda
2025/05/27 12:00:00 Nasłuchiwanie na porcie TCP: 8089
2025/05/27 12:00:00 Serwer uruchomiony: http://localhost:8089
```

### d. Liczba warstw i rozmiar obrazu

```bash
# Rozmiar obrazu
docker images nazariil/weather-app:v1.0

# Liczba warstw
docker inspect nazariil/weather-app:v1.0 --format='{{len .RootFS.Layers}}'

# Szczegóły warstw
docker history nazariil/weather-app:v1.0
```

Obraz oparty na `scratch` posiada **3 warstwy**: certyfikaty SSL, binarny plik aplikacji, szablony HTML.

---

## Podgląd aplikacji

![Podgląd aplikacji](Część%20obowiązkowa/pogoda.png)

---

## Część dodatkowa

Szczegóły w pliku [`Część nieobowiązkowa/zadanie1_dod.md`](Część%20nieobowiązkowa/zadanie1_dod.md).
