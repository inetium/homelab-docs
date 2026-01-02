# Gitea – serwer Git

## Rola w homelabie

Gitea pełni rolę lekkiego, samodzielnego serwera Git z webowym interfejsem (odpowiednik „mini GitLaba” / GitHuba) dla:

- repozytoriów konfiguracyjnych (np. `homelab-docs`, pliki Docker Compose),
- projektów, skryptów i narzędzi związanych z homelabem,
- ewentualnie osobistych projektów programistycznych.

Dostęp do Gitei jest realizowany przez Traefika pod adresem:

- `http://gitea.lab`

---

## Architektura Gitei

Gitea składa się z dwóch głównych kontenerów:

- **`gitea`** – aplikacja Gitea (interfejs WWW, API, obsługa repozytoriów),
- **`gitea-db`** – baza danych PostgreSQL.

### Sieci Dockera

Gitea jest podpięta do dwóch sieci:

- `gitea_net` – sieć wewnętrzna do komunikacji z bazą danych,
- `proxy` – sieć współdzielona z Traefikiem, aby wystawić Giteę jako `gitea.lab`.

---

## Docker Compose – aktualna konfiguracja

Plik: `/srv/docker/gitea/compose.yml`:

```yaml
version: "3.8"

services:
  db:
    image: postgres:16
    container_name: gitea-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: gitea
      POSTGRES_USER: gitea
      POSTGRES_PASSWORD: "Gitea_2024!_db"
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - gitea_net

  gitea:
    image: gitea/gitea:1.22-rootless
    container_name: gitea
    restart: unless-stopped
    environment:
      USER_UID: 1000          # UID użytkownika 'marcin'
      USER_GID: 1003          # GID użytkownika 'marcin'

      # Konfiguracja bazy danych
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432
      GITEA__database__NAME: gitea
      GITEA__database__USER: gitea
      GITEA__database__PASSWD: "Gitea_2024!_db"

      # Konfiguracja serwera HTTP/URL
      GITEA__server__ROOT_URL: "http://gitea.lab/"
      GITEA__server__DOMAIN: "gitea.lab"
      GITEA__server__HTTP_PORT: 3000

    volumes:
      - ./gitea:/var/lib/gitea

    # Tymczasowy bezpośredni dostęp (jeśli potrzebny)
    # ports:
    #   - "3002:3000"           # http://10.10.20.10:3002 (czasowy dostęp)
    #   - "2222:22"             # SSH do repo (opcjonalnie, gdy Gitea SSH jest używane)

    depends_on:
      - db

    networks:
      - gitea_net
      - proxy

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.gitea.rule=Host(`gitea.lab`)"
      - "traefik.http.routers.gitea.entrypoints=web"
      - "traefik.http.services.gitea.loadbalancer.server.port=3000"

networks:
  gitea_net:
    name: gitea_net
  proxy:
    external: true
```
## Dane i uprawnienia
### Ścieżki danych
- Dane Gitei:

  `./gitea` → montowane do `/var/lib/gitea` w kontenerze Gitei.
- Dane Postgresa:
  
  `./postgres` → montowane do `/var/lib/postgresql/data` w kontenerze bazy.

Dzięki temu:

- Można bezpiecznie aktualizować kontenery (obrazy) bez utraty danych.
- Backup Gitei sprowadza się m.in. do backupu katalogów `gitea/` i `postgres/`.
### Uprawnienia (rootless Gitea)
Gitea działa w trybie **rootless**, z UID/GID użytkownika marcin:

- `USER_UID: 1000`
- `USER_GID: 1003`
  
Przed pierwszym uruchomieniem zostały poprawione uprawnienia:

```bash
chown -R 1000:1003 gitea postgres
```
To zapewnia, że kontener Gitei ma prawo zapisu do katalogów z danymi.

## Połączenie z bazą Postgres
Konfiguracja bazy danych:

- Typ: `PostgreSQL`
- Host: `db:5432` (serwis w docker-compose, sieć `gitea_net`)
- Baza: `gitea`
- Użytkownik: `gitea`
- Hasło: `Gitea_2024!_db`
  
Te same parametry są ustawione:

- w zmiennych środowiskowych kontenera Gitei,

- oraz były podane podczas wstępnej konfiguracji Gitei w przeglądarce.
### Konfiguracja URL i domeny
Gitea jest dostępna przez Traefika jako `http://gitea.lab`.

Dlatego w konfiguracji serwera Gitei:

- `ROOT_URL` = `http://gitea.lab/`
- `DOMAIN` = `gitea.lab`
- `HTTP_PORT` = `3000` (port wewnątrz kontenera)

Dzięki temu:

- wszystkie linki generowane przez Giteę (np. do repozytoriów, clona, avatarów) używają `http://gitea.lab/`,
- nie ma problemu z linkami wskazującymi na `IP 10.10.20.10:3002`.
## Wystawienie przez Traefika (gitea.lab)
Traefik wykrywa kontener Gitei przez provider Docker i sieć proxy. 

Etykiety:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.gitea.rule=Host(`gitea.lab`)"
  - "traefik.http.routers.gitea.entrypoints=web"
  - "traefik.http.services.gitea.loadbalancer.server.port=3000"
```
Wymagania:

- Gitea musi być w sieci proxy (wspólnej z Traefikiem).
- W Pi-hole musi istnieć rekord DNS:
  `gitea.lab` → `10.10.20.10`.

Przepływ:

1. Klient wpisuje `http://gitea.lab`.
2. DNS (Pi-hole) rozwiązuje `gitea.lab` na `10.10.20.10`.
3. Traefik odbiera żądanie na porcie 80, widzi `Host: gitea.lab`.
4. Router `gitea` kieruje ruch do kontenera Gitei na porcie 3000.
5. Użytkownik widzi interfejs Gitei.
## Korzystanie z Gitei
### Dostęp przez przeglądarkę
- URL: `http://gitea.lab`
- Logowanie:
  - użytkownik admina skonfigurowany podczas instalacji (np. marcin),
  - możliwość zakładania kolejnych kont, jeśli potrzeba.
Klonowanie repozytoriów (HTTP)
Przykłady:


```bash
git clone http://gitea.lab/marcin/homelab-docs.git
git clone http://gitea.lab/marcin/jakies-inne-repo.git
```
Daje to możliwość:

- edycji dokumentacji homelabu (`homelab-docs`) na laptopie (VS Code),
- trzymania pozostałych konfiguracji w Git (np. pliki Docker Compose, skrypty).
### SSH do repozytoriów (opcjonalnie)
Jeśli skonfigurujesz port SSH Gitei:

- w docker-compose można dodać np.:
```yaml
ports:
  - "2222:22"
```
- w ustawieniach Gitei ustawić port SSH na 2222,
- dodać klucze SSH do swojego konta w Gitei.
- 
Klonowanie przez SSH (przykład):

```bash
git clone ssh://git@gitea.lab:2222/marcin/homelab-docs.git
```
*(Na razie można zostać przy HTTP, jeśli SSH nie jest potrzebne.)*

## Typowe zastosowania w homelabie
- **Dokumentacja**
  
  Repo: `homelab-docs` → źródło dla `docs.lab` (Material for MkDocs).
- **Infra as Code**

  Pliki `compose.yml` dla Traefika, Portainera, Gitei, monitoringu itd. w osobnych repozytoriach.
- **Backup konfiguracji**
  Eksporty konfiguracji MikroTika, pliki konfiguracyjne Pi-hole, Traefika, Prometheusa, Grafany – trzymane w Gicie.

Dzięki temu:

- masz wersjonowaną historię zmian konfiguracji,
- możesz łatwo cofnąć się do poprzednich wersji,
- możesz klonować konfigurację na inne maszyny (np. nowy serwer w homelabie).
## Procedura restartu / aktualizacji Gitei
Jeśli zmienisz `compose.yml` lub chcesz zaktualizować obraz Gitei / Postgresa:

```bash
cd /srv/docker/gitea

# Pobranie nowych wersji obrazów (opcjonalnie)
docker compose pull

# Restart z aktualizacją
docker compose up -d
```
Przed większymi zmianami warto:

- wykonać backup katalogów `./gitea` i `./postgres`,
- ewentualnie zrzut bazy Postgresa (np. `pg_dump`).
## Podsumowanie
- Gitea działa jako centralny serwer Git w homelabie.
- Jest podpięta:
  - do Postgresa (sieć `gitea_net`),
  - do Traefika (sieć `proxy`, host `gitea.lab`).
- Dane są trzymane w wolumenach (`./gitea`, `./postgres`).
- Integracja z Pi-hole i Traefikiem zapewnia prosty dostęp `http://gitea.lab`.
- W połączeniu z VS Code na Windows tworzy wygodny pipeline:
  - edycja lokalna → `git push` → Gitea → `git pull` na rpi5.