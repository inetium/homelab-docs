# Docker i Docker Compose

## Rola Dockera w homelabie

Docker jest podstawą uruchamiania większości usług w homelabie:

- Traefik (reverse proxy)
- Portainer (zarządzanie Dockerem)
- Gitea (serwer Git)
- Pi-hole (DNS + blokowanie reklam)
- Monitoring (Prometheus, Grafana, Loki, Uptime Kuma, itp.)
- Dokumentacja (MkDocs / Material – `docs.lab`)

Dzięki Dockerowi:

- każda usługa działa w swoim kontenerze,
- konfiguracja jest deklaratywna (pliki `compose.yml`),
- łatwo jest przenosić, backupować i odtwarzać usługi.

---

## Instalacja Dockera na Ubuntu 25.10

Docker został zainstalowany **z oficjalnego repozytorium Dockera**, a nie z paczki z Ubuntu (która bywa przestarzała).

### Krok 1 – usunięcie ewentualnych starych pakietów

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```
### Krok 2 – wymagane pakiety
```bash
sudo apt update
sudo apt install ca-certificates curl gnupg
```
### Krok 3 – dodanie oficjalnego repo Dockera
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Zastąp $(. /etc/os-release; echo "$VERSION_CODENAME") odpowiednim codename (25.10)
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release; echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### Krok 4 – instalacja Dockera i Compose
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
### Krok 5 – dodanie użytkownika do grupy docker
```bash
sudo usermod -aG docker marcin
# wylogowanie / zalogowanie, żeby grupa zadziałała
```
Dzięki temu można używać docker i docker compose bez sudo.

## Ogólna struktura `/srv/docker/`
Wszystkie usługi dockera w homelabie są trzymane w jednym katalogu:

```text
/srv/docker/
├── traefik/
├── portainer/
├── gitea/
├── monitoring/
├── pihole/
└── docs/
```
Zasada:

- **jedna usługa = jeden katalog**,
- w każdym katalogu:
  - własny `compose.yml`,
  - dane specyficzne dla usługi (volumes),
  - ewentualne dodatkowe pliki konfiguracyjne.
  
Przykładowo:

```text
/srv/docker/traefik/
  └── compose.yml

/srv/docker/portainer/
  ├── compose.yml
  └── data/

/srv/docker/gitea/
  ├── compose.yml
  ├── gitea/
  └── postgres/

/srv/docker/monitoring/
  ├── compose.yml
  ├── prometheus/
  ├── grafana/
  └── ...
```
## Kluczowe katalogi usług
`/srv/docker/traefik/`
- `compose.yml` – konfiguracja Traefika jako centralnego reverse proxy.
- Używa sieci proxy (external).
- Wystawia usługi `*.lab` (Portainer, Gitea, Docs, Grafana, itd.).
  
Szczegóły: zobacz `Usługi → Traefik`.

---
`/srv/docker/portainer/`
- `compose.yml`– uruchamia Portainer CE.
- Montuje socket Dockera: `/var/run/docker.sock`.
- Dane Portainera: `./data`.

Portainer jest dostępny pod:
- `http://portainer.lab` (za Traefikiem).
---
`/srv/docker/gitea/`
- `compose.yml` – uruchamia:
  - Postgresa (`gitea-db`),
  - Giteę (`gitea`).
- Volumes:
  - `./gitea` → dane Gitei (`/var/lib/gitea`),
  - `./postgres` → dane Postgresa (`/var/lib/postgresql/data`).
Sieci:

- `gitea_net` – wewnętrzna (Gitea ↔ Postgres),
- `proxy` – do wystawienia `gitea.lab` przez Traefika.
  
Szczegóły: zobacz `Usługi → Gitea`.

---
`/srv/docker/monitoring/`

*(Docelowy katalog, jeśli nie jest jeszcze w pełni skonfigurowany)*

- `compose.yml` – stack monitoringu:
  - **Prometheus** – metryki,
  - **Grafana** – wizualizacja,
  - (docelowo) **Loki/Promtail** – logi,
  - **Uptime Kuma** – monitoring dostępności.
- Struktura przykładowa:
```text
/srv/docker/monitoring/
  ├── compose.yml
  ├── prometheus/
  │   ├── prometheus.yml
  │   └── data/
  ├── grafana/
  │   ├── data/
  │   └── provisioning/
  ├── loki/
  └── uptime-kuma/
```
Sieci:

- `monitoring` – wewnętrzna sieć dla komponentów monitoringu,
- `proxy` – do wystawienia grafana.lab, kuma.lab, itp.
---
`/srv/docker/pihole/`

- `compose.yml` – uruchamia Pi-hole jako kontener Dockera.
- Porty:
  - `53/udp` i `53/tcp` – DNS (mapowane na `10.10.20.10`),
  - port panelu WWW (80 lub inny, jeśli koliduje z Traefikiem).
- Volumes:
  - konfiguracje i lists: np. `./etc-pihole`, `./etc-dnsmasq.d`.
  
Rola:

- główny DNS sieci (`10.10.20.10`),
- lokalne rekordy `*.lab`,
- filtr reklam.
  
Szczegóły: `Usługi → DNS`.

---
`/srv/docker/docs/`
- `compose.yml` – uruchamia MkDocs/Material (kontener `squidfunk/mkdocs-material`).
- Volumes:
  - `./mkdocs.yml` → konfiguracja dokumentacji,
  - `./docs/` → pliki `.md` (treść dokumentacji).
- Sieć:
  - `proxy` – aby wystawiać `docs.lab` przez Traefika.
  
Rola:

- serwowanie dokumentacji homelabu pod `http://docs.lab`.
  
## Sieci Dockera
W homelabie używane są różne sieci Dockera:

## `proxy` **(external)**
- Główna, współdzielona sieć dla usług wystawianych przez Traefika.
- Podłączone są do niej m.in.:
  - `traefik`,
  - `portainer`,
  - `gitea`,
  - `docs` (MkDocs),
  - `grafana, prometheus` (opcjonalnie),
  - inne usługi HTTP/HTTPS.
  
Tworzona ręcznie (jednorazowo):

```bash
docker network create proxy
```
---

## `gitea_net`
- Sieć prywatna pomiędzy:
  - Giteą,
  - bazą Postgresa Gitei.
  
Zapewnia izolację ruchu bazy danych od reszty usług.  
Zdefiniowana w `compose.yml` Gitei:

```yaml
networks:
  gitea_net:
    name: gitea_net
```
---

## `monitoring` **(przykładowa)**
- Sieć prywatna dla:
  - Prometheusa,
  - Grafany,
  - Lokiego/Promtaila,
  - ewentualnie exporterów.
  
Definiowana w `compose.yml` monitoringu:

```yaml
networks:
  monitoring:
    driver: bridge
```
---

## Zasady organizacji Dockera w homelabie
1. **Jedna usługa / stack = jeden katalog w `/srv/docker/`**
    - np. `traefik, portainer, gitea, monitoring, pihole, docs`.
2. **Każdy katalog ma własny `compose.yml`**
    - łatwy restart/aktualizacja pojedynczej usługi:
```bash
cd /srv/docker/gitea
docker compose up -d
```
3. **Dane trzymane w podkatalogach**
    - np. `./gitea, ./postgres, ./grafana/data, ./prometheus/data, ./data` (Portainer),
    - to ułatwia backup (wersja „docker level backup”).
4. **Wspólna sieć proxy dla usług HTTP za Traefikiem**
    - wszystkie usługi dostępne jako *.lab siedzą w proxy.
5. **Osobne sieci prywatne dla specyficznych powiązań**
    - np. `gitea_net` dla Gitei i jej bazy,
    - `monitoring` dla Prometheusa/Grafany/Lokiego.
6. **DNS i Traefik jako „kręgosłup”**
    - Pi-hole (DNS) daje nazwy `*.lab`,
    - Traefik routuje HTTP/HTTPS na podstawie hostów.
---
## Aktualizacje i restarty – podejście 
### Aktualizacja pojedynczej usługi 
Dla każdej usługi:

```bash
cd /srv/docker/<usługa>
docker compose pull      # pobierz nowe wersje obrazów
docker compose up -d     # uruchom z nowymi obrazami
```
Przykłady:

  - aktualizacja Traefika:
```bash
cd /srv/docker/traefik
docker compose pull
docker compose up -d
```
  - aktualizacja Gitei:
```bash
cd /srv/docker/gitea
docker compose pull
docker compose up -d
```
  - aktualizacja monitoringu:
```bash
cd /srv/docker/monitoring
docker compose pull
docker compose up -d
```
### Restart usługi (bez aktualizacji)
```bash
cd /srv/docker/<usługa>
docker compose restart
```
Przykłady:

  - restart dokumentacji (MkDocs), gdy zmienia się `mkdocs.yml`:
```bash
cd /srv/docker/docs
docker compose restart docs
```
  - restart Traefika po zmianie parametrów w compose.yml:
```bash
cd /srv/docker/traefik
docker compose restart traefik
```
### Zależność od zmian konfiguracji
- #### Zmiany tylko w treści (np. `.md` w MkDocs)
→ najczęściej wystarczy `git pull`, ewentualnie `docker compose restart <usługa>`, gdy auto‑reload nie zadziała.
- #### Zmiany w plikach `compose.yml` lub zmiennych środowiskowych
→ zalecane docker `compose up -d` (czasem z `pull`, gdy zmieniasz wersję obrazów).
- #### Zmiany w strukturze sieci / volume’ów
→ dobrze jest zatrzymać stack (`docker compose down`), potem `up -d` z nową konfiguracją, po wcześniejszym backupie danych.

---

## Backup i odtwarzanie

Podejście do backupu na poziomie Dockera:

1. **Konfiguracja / deklaracja:**
    - pliki `compose.yml` w `/srv/docker/*`,
    - najlepiej trzymać je w Gicie (np. w Gitei).
2. **Dane usług:**
  - katalogi typu:
    - `/srv/docker/gitea/gitea`,
    - `/srv/docker/gitea/postgres`,
    - `/srv/docker/monitoring/prometheus/data`,
    - `/srv/docker/monitoring/grafana/data`,
    - `/srv/docker/pihole/etc-pihole`, itp.
3. **Procedura odtworzenia (w uproszczeniu):**
    - instalacja Dockera (jak wyżej),
    - odtworzenie struktury `/srv/docker/` (pliki i katalogi z backupu),
    - docker `network create proxy` (i inne sieci, jeśli trzeba),
    - po kolei:
```bash
cd /srv/docker/traefik     && docker compose up -d
cd /srv/docker/pihole      && docker compose up -d
cd /srv/docker/gitea       && docker compose up -d
cd /srv/docker/monitoring  && docker compose up -d
cd /srv/docker/docs        && docker compose up -d
# itd.
```
Szczegóły i scenariusze backupu opisuję szerzej w sekcji:
`Kopie zapasowe`.

## Podsumowanie
- Docker jest kluczową warstwą uruchamiania usług w homelabie.
- Wszystkie usługi są zorganizowane w `/srv/docker/` według zasady:
    - jedna usługa / stack = jeden katalog,
    - każdy katalog ma własny `compose.yml` i katalogi z danymi.
- Sieć proxy spina usługi HTTP za Traefikiem, a prywatne sieci (`gitea_net, monitoring`, itp.) zapewniają izolację.
- Aktualizacje odbywają się per‑usługa przy pomocy:
    - `docker compose pull`,
    - `docker compose up -d`,
    - `docker compose restart`.
- Konfiguracje Dockera warto trzymać w Gitea, co ułatwia wersjonowanie i odtwarzanie całego środowiska.
- 
Docker razem z Traefikiem i Pi-hole tworzą fundament homelabu – reszta usług (Gitea, monitoring, dokumentacja itd.) to tylko „klocki” ułożone na tej bazie.