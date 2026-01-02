# Grafana – wizualizacja metryk

## Rola w homelabie

Grafana pełni rolę **centralnego panelu wizualizacji** dla wszystkich metryk i logów w homelabie.

Jej główne zadania:

- prezentacja metryk z **Prometheusa** na czytelnych dashboardach,
- (docelowo) integracja z **Loki** (logi) i innymi źródłami danych,
- tworzenie widoków:
  - „zdrowie” homelabu,
  - obciążenie serwera (Raspberry Pi 5),
  - stan kontenerów Dockera i usług za Traefikiem,
  - statystyki ruchu HTTP.

W praktyce:

- Prometheus zbiera i trzyma liczby,  
- Grafana pokazuje je w sposób, który ma sens dla człowieka.

---

## Lokalizacja w homelabie

- Serwer: **Raspberry Pi 5** (`10.10.20.10`).
- Katalog na docker-compose:     `/srv/docker/monitoring/`.
- Dostęp przez przeglądarkę (docelowo za Traefikiem):

  - `http://grafana.lab`

---

## Podstawowa architektura

Grafana działa jako kontener Dockera, który:

- nasłuchuje na porcie HTTP (domyślnie `3000` w kontenerze),
- trzyma konfigurację i dane (użytkownicy, datasources, dashboardy) w wolumenie,
- łączy się do różnych **Data Sources**:
  - w pierwszej kolejności: **Prometheus**,
  - (docelowo) **Loki** dla logów,
  - inne, jeśli kiedyś dodasz (np. InfluxDB, MySQL itd.).

Schemat połączeń:

1. Prometheus zbiera metryki z exporterów (host, Docker, Traefik, inne).
2. Grafana jako Data Source ma skonfigurowany URL do Prometheusa.
3. Dashboardy w Grafanie odpytywują Prometheusa i rysują wykresy.

---

## Typowy docker-compose dla Grafany (schemat)

Przykładowy (schematyczny) fragment `docker-compose.yml` dla Grafany:

```yaml
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: "admin"
      GF_SECURITY_ADMIN_PASSWORD: "admin"  # ZMIENIĆ!
    volumes:
      - ./grafana/data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - proxy
      - monitoring
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.lab`)"
      - "traefik.http.routers.grafana.entrypoints=web"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

networks:
  proxy:
    external: true
  monitoring:
    driver: bridge
```
Założenia:

- Grafana jest w sieci monitoring razem z Prometheusem i innymi komponentami monitoringu.
- Jest też w sieci proxy, aby Traefik mógł ją wystawić jako `grafana.lab`.
- Dane Grafany są trzymane w wolumenie `./grafana/data`.

## Wystawienie przez Traefika (grafana.lab)
Etykiety Traefika (labels) dla Grafany:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.grafana.rule=Host(`grafana.lab`)"
  - "traefik.http.routers.grafana.entrypoints=web"
  - "traefik.http.services.grafana.loadbalancer.server.port=3000"
  ```
Wymagane elementy:

- Grafana podpięta do sieci proxy.
- W Pi-hole rekord DNS: `grafana.lab → 10.10.20.10`.

Przepływ:

1. Klient wpisuje `http://grafana.lab`.
2. DNS (Pi-hole) rozwiązuje `grafana.lab` na `10.10.20.10`.
3. Traefik odbiera żądanie na porcie 80, widzi Host: `grafana.lab`.
4. Router `grafana` przekazuje ruch do kontenera Grafany (port 3000).
5. Użytkownik widzi panel logowania do Grafany.
   
## Integracja z Prometheusem (data source)
Po pierwszym zalogowaniu do Grafany (domyślnie `admin / admin`, później warto zmienić hasło):

1. Przechodzisz do: `Configuration → Data sources`.
2. Dodajesz nowe źródło danych: `Type: Prometheus`.
3. Ustawiasz URL Prometheusa: jeśli Grafana i Prometheus są w tej samej sieci Dockerowej (`monitoring`), URL może być: `http://prometheus:9090`, jeśli inaczej – zgodnie z Twoim `docker-compose.yml`.
4. Zapisujesz i testujesz połączenie (`Save & test`).
   
Po poprawnej konfiguracji możesz używać Prometheusa jako źródła danych dla dashboardów.

## Przykładowe dashboardy
Typowe dashboardy, które mają sens w tym homelabie:

1. „Homelab overview”
- status Raspberry Pi 5:
  - CPU, RAM, swap,
  - obciążenie (load average),
  - temperatura, dyski,
  - statystyki ruchu przez Traefika:
  - requesty na usługę (portainer, gitea, docs itp.),
  - błędy 4xx/5xx,
- status kluczowych kontenerów:
  - uptime,
  - ilość restartów.
2. „Raspberry Pi 5 – host details”
- CPU usage per core,
- RAM / swap usage,
- I/O dysków,
- sieć (przepływność, błędy),
- temperatura CPU (jeśli eksportowana).
3. „Docker containers”
- zużycie CPU / RAM per kontener,
- liczba restartów,
- wykorzystanie sieci przez kontenery,
- stan (`up / down`).
4. „Traefik / HTTP”
- liczba requestów w czasie (per usługa: Portainer, Gitea, Docs, Grafana, itd.),
- kody odpowiedzi (2xx, 4xx, 5xx),
- średnie opóźnienie (latency).
  
Wiele gotowych dashboardów można znaleźć w Grafana Dashboards (oficjalna biblioteka), importując je po ID.

## Integracja z Lokim (na przyszłość)
Docelowo Grafana może też służyć do przeglądania logów:

- **Loki** – system logów (podobny do Prometheusa, ale dla tekstu),
- **Promtail** – agent, który zbiera logi i wysyła do Lokiego.
- 
Wtedy w Grafanie:

- dodajesz drugie źródło danych typu Loki,
- możesz mieć zakładki:
  - metryki (Prometheus),
  - logi (Loki),
- i łączyć logi z metrykami (np. klikając punkt na wykresie, zobaczyć logi z tego okresu).
## Uwierzytelnianie i dostęp
Na start:

- Grafana ma domyślnego admina (`admin / admin`).
- Po pierwszym logowaniu należy zmienić hasło admina.
 
Możliwe kierunki rozwoju:

- integracja z zewnętrznym systemem auth (LDAP, OAuth, itd.) – raczej „overkill” dla domowego labu,
- tworzenie osobnych kont użytkowników (np. readonly dla domowników).
## Backup i persystencja
Dane Grafany:

- konfiguracja, dashboardy, dane użytkowników: w wolumenie `./grafana/data`.
  
Przy backupie homelabu warto obejmować:

- `./grafana/data` (konfiguracja i dashboardy),
- pliki provisioning (`./grafana/provisioning`), jeśli z nich korzystasz.
  
W razie potrzeby Grafanę można łatwo przenieść na inny host:

- skopiować wolumen,
- odpalić nowy kontener Grafany z tym samym `GF_SECURITY_ADMIN_USER/PASSWORD`.
## Podsumowanie
- Grafana jest „oknem na świat” metryk i logów w homelabie.
- Działa w Dockerze na rpi5, zwykle w katalogu `/srv/docker/monitoring/`.
- Jest wystawiona za Traefikiem jako `http://grafana.lab`.
- Łączy się z Prometheusem jako głównym źródłem danych.
- Umożliwia budowę:
  - dashboardów dla hosta rpi5,
  - monitoringu kontenerów,
  - statystyk ruchu HTTP przez Traefika.
- W przyszłości może być spięta również z Lokim (logi) i innymi źródłami danych.
- 
Grafana jest razem z Prometheusem kluczowym elementem sekcji:
`Usługi → Monitoring`.