# Traefik – reverse proxy

## Rola w homelabie

Traefik działa jako centralny **reverse proxy** dla wszystkich usług HTTP/HTTPS w homelabie.

Dzięki niemu:

- Wszystkie usługi są dostępne pod „ludzkimi" nazwami `*.lab` (np. `portainer.lab`, `gitea.lab`, `docs.lab`).
- Nie trzeba pamiętać portów (np. `10.10.20.10:9000`, `10.10.20.10:3002`).
- Traefik automatycznie wykrywa nowe kontenery Dockera (dzięki providerowi Docker) i tworzy dla nich routery na podstawie **etykiet** (labels).
- W przyszłości łatwo dodać certyfikaty SSL (Let's Encrypt) dla HTTPS.

---

## Lokalizacja i podstawowe założenia

- Serwer: **Raspberry Pi 5** (`10.10.20.10`)
- Katalog: `/srv/docker/traefik/`
- Plik konfiguracji: `compose.yml`
- Sieć Docker: `proxy` (external, współdzielona z innymi usługami)
- Porty:
  - `80` – HTTP (entrypoint `web`)
  - `443` – HTTPS (entrypoint `websecure`, na razie bez certyfikatów)
  - `8085` – Dashboard Traefika (dostępny z LAN)

---

## Docker Compose – aktualny `compose.yml`

Plik `/srv/docker/traefik/compose.yml`:

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped

    command:
      # Entrypoint HTTP (port 80 w kontenerze)
      - "--entrypoints.web.address=:80"

      # Entrypoint HTTPS (port 443 w kontenerze)
      - "--entrypoints.websecure.address=:443"

      # Provider Docker
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"

      # Dashboard / API
      - "--api.dashboard=true"
      - "--api.insecure=true"           # dashboard bez auth, na :8080 w kontenerze

      # Logi Traefika
      - "--log.level=INFO"

    ports:
      - "80:80"                         # HTTP z LAN
      - "443:443"                       # HTTPS z LAN (później certy)
      - "8085:8080"                     # dashboard: http://10.10.20.10:8085

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

    networks:
      - proxy

networks:
  proxy:
    external: true
```
### Kluczowe parametry

- `--providers.docker=true` – Traefik automatycznie wykrywa kontenery Dockera.
- `--providers.docker.exposedbydefault=false` – kontenery **nie są** automatycznie wystawiane; trzeba dodać etykietę `traefik.enable=true`.
- `--api.dashboard=true` + `--api.insecure=true` – włącza dashboard Traefika bez uwierzytelniania (dostępny tylko z LAN).
- Sieć `proxy` jest **external** – musi być utworzona ręcznie przed uruchomieniem Traefika:

  ```bash
  docker network create proxy
  ```

  ---

### Dostęp do dashboardu Traefika

Dashboard Traefika jest dostępny pod adresem:

  `http://10.10.20.10:8085`
  
W dashboardzie widać aktywne routery (HTTP/HTTPS), usługi (backendy), middlewares oraz status certyfikatów. Jest to podstawowe narzędzie do sprawdzania, czy Traefik poprawnie „widzi” nowo dodane kontenery.

---
### Jak traefik routuje ruch?

Traefik działa w oparciu o etykiety (labels) dodawane do kontenerów Dockera.

Podstawowy schemat:

1. Klient w LAN wpisuje w przeglądarce np. `http://portainer.lab`.
2. DNS (Pi-hole) zwraca IP rpi5: `10.10.20.10`.
3. Przeglądarka łączy się z `10.10.20.10:80` z nagłówkiem Host: `portainer.lab`.
4. Traefik znajduje router z regułą Host(`portainer.lab`) i przekierowuje ruch do właściwego kontenera.
---
### Przykłady etykiet dla usług

#### Portainer (`portainer.lab`)
Plik `/srv/docker/portainer/compose.yml`:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.portainer.rule=Host(`portainer.lab`)"
  - "traefik.http.routers.portainer.entrypoints=web"
  - "traefik.http.services.portainer.loadbalancer.server.port=9000"
```
#### MkDocs (`docs.lab`)
Plik `/srv/docker/docs/compose.yml` :

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.docs.rule=Host(`docs.lab`)"
  - "traefik.http.routers.docs.entrypoints=web"
  - "traefik.http.services.docs.loadbalancer.server.port=8000"
```
#### Gitea (`gitea.lab`)
Aby Gitea działała poprawnie za Traefikiem, musi być w sieci `proxy` oraz mieć ustawiony `ROOT_URL`:
```yaml
# Fragment labels w compose.yml Gitei
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.gitea.rule=Host(`gitea.lab`)"
  - "traefik.http.routers.gitea.entrypoints=web"
  - "traefik.http.services.gitea.loadbalancer.server.port=3000"
```


## Dodawanie nowych usług

Aby dodać nową usługę (np. Grafana, Uptime Kuma), należy:

1. Dodać kontener do sieci `proxy`.
2. Dodać etykiety `traefik.*` (zgodnie z przykładami powyżej).
3. Dodać rekord DNS w Pi-hole: `<nazwa>.lab → 10.10.20.10`.
4. Uruchomić kontener (`docker compose up -d`).

## Podsumowanie
- Traefik to centralny punkt wejścia dla usług HTTP w homelabie.
- Obsługuje domeny `*.lab` bez konieczności pamiętania numerów portów.
- Konfiguracja odbywa się dynamicznie poprzez etykiety w plikach `compose.yml`.