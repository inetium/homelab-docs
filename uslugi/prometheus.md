# Prometheus – monitoring metryk

## Rola w homelabie

Prometheus pełni rolę **głównego zbieracza metryk** w homelabie.  
Jego zadania:

- okresowo odpytuje różne **eksportery** (Prometheus exporters),
- zbiera metryki dotyczące:
  - zasobów serwerów (CPU, RAM, dysk, sieć),
  - kontenerów Dockera,
  - usług (np. Traefik),
- przechowuje dane czasowe (time series),
- udostępnia je **Grafanie** jako źródło danych do dashboardów.

W skrócie:

> Prometheus = silnik zbierania metryk i „baza danych czasowych”  
> Grafana = panel do wizualizacji tych metryk

---

## Lokalizacja w homelabie

- Serwer: **Raspberry Pi 5** (`10.10.20.10`) – ten sam, na którym działa Docker, Traefik, Pi-hole itd.
- Katalog na docker-compose:  
  `TODO` – np. `/srv/docker/monitoring/`  
  *(u mnie monitoring przewidziany jest właśnie tam, z Prometheusem, Grafaną, Loki, Uptime Kuma, itd.)*
- Prawdopodobny dostęp (docelowo za Traefikiem, choć rzadko zagląda się do UI Prometheusa bezpośrednio):
  - np. `http://prometheus.lab` → panel Prometheusa,
  - ale w praktyce głównie korzysta się z niego pośrednio przez Grafanę.

---

## Co zbiera Prometheus?

Typowe źródła metryk w takim homelabie:

- **Raspberry Pi 5 / host**:
  - exporter typu `node_exporter` (CPU, RAM, dyski, sieć, load, itp.),
- **Docker / kontenery**:
  - np. `cAdvisor` lub `docker_exporter` (zużycie zasobów przez kontenery),
- **Traefik**:
  - statystyki ruchu HTTP, liczba requestów, kody odpowiedzi, latency (jeśli włączona jest integracja z Prometheusem),
- **Inne usługi** (opcjonalnie):
  - np. eksportery do MikroTika, Pi-hole, Postgresa itp., jeżeli je kiedyś dodasz.

Dalej metryki są trzymane w Prometheusie i odpytywane przez Grafanę, gdzie buduje się dashboardy.

---

## Podstawowa architektura Prometheusa

Prometheus działa jako jeden kontener Dockera, który:

- nasłuchuje na porcie HTTP (zwykle `9090`),
- ma w środku plik konfiguracyjny `prometheus.yml`,
- w tym pliku ma zdefiniowane **jobs** (zestawy endpointów do scraping’u).

Przykładowa logika (uproszczona, schematyczna):

- `job: node_exporter`:
  - `targets: [ 'rpi5:9100' ]`
- `job: traefik`:
  - `targets: [ 'traefik:9100' ]` lub endpoint metryk Traefika
- `job: docker`:
  - `targets: [ 'cadvisor:8080' ]`

*(to przykład, niekoniecznie 1:1 z Twoją konfiguracją – konkretne porty i nazwy będą w Twoim `prometheus.yml`)*

---

## Typowy docker-compose dla Prometheusa (schemat)

Przykładowy (schematyczny) fragment `docker-compose.yml` dla Prometheusa w katalogu monitoringu:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    ports:
      - "9090:9090"    # tymczasowo, do debugowania
    networks:
      - proxy          # opcjonalnie, jeśli wystawiasz przez Traefika
      - monitoring     # wewnętrzna sieć monitoringu

networks:
  proxy:
    external: true
  monitoring:
    driver: bridge
```
Docelowo Prometheus mógłby być dostępny np. pod:

`http://prometheus.lab` → przez Traefika,
  
ale w praktyce:

- UI Prometheusa jest mało używany do codziennej pracy,
- większość rzeczy i tak ogląda się w Grafanie.
## Integracja z Grafaną
Prometheus jest skonfigurowany w Grafanie jako Data Source typu `Prometheus`.

W Grafanie:

- dodajesz nowy data source,
- URL: np. `http://prometheus:9090` (z poziomu sieci Dockera),
- wybierasz namespace / labele itd. dla dashboardów.
- 
Dzięki temu możesz:

- tworzyć dashboardy z metryk:
  - hosta rpi5,
  - kontenerów,
  - usług (Traefik, itp.),
- ustawiać alerty (np. CPU > 90%, mało miejsca na dysku, kontener nie odpowiada).
- 
Szczegóły integracji po stronie Grafany opisuję w osobnej sekcji:
`Usługi → Grafana`.

## Powiązanie z innymi komponentami monitoringu
W docelowym setupie monitoringowym:

- **Prometheus** – zbiera metryki (time series).
- **Grafana** – wizualizuje metryki z Prometheusa (dashboardy).
- **Loki + Promtail** – zbierają logi (wydarzenia tekstowe, a nie liczby).
- **Uptime Kuma** – monitoruje dostępność usług (HTTP/TCP/ping) i wysyła powiadomienia.

Prometheus skupia się na:

- **„jak dużo”** – obciążenie CPU, RAM, dysków, ilość requestów, błędów, itd.
- **w czasie** – z możliwością trendów, wykresów historycznych, wykrywania anomalii.

## Typowe zastosowania Prometheusa w homelabie
- Monitorowanie stanu Raspberry Pi 5:
  - CPU, RAM, temperatura, obciążenie, zużycie dysku.
- Monitorowanie kontenerów Dockera:
  - zużycie zasobów przez poszczególne usługi,
  - liczba restartów, uptime.
- Statystyki Traefika:
  - ilość requestów HTTP na poszczególne usługi,
  - kody odpowiedzi (2xx, 4xx, 5xx),
  - latency.
- Dane do dashboardów typu:
  - „Health homelabu”,
  - „Obciążenie serwera”,
  - „Ruch HTTP przez Traefika”.
## Podsumowanie
- **Prometheus** jest centralnym silnikiem zbierania metryk w homelabie.
- Działa w Dockerze na rpi5, w katalogu `/srv/docker/monitoring/`.
- Zbiera dane z:
  - eksportera hosta (`node_exporter`),
  - eksportera Dockera (np. `cAdvisor`),
  - Traefika i innych usług (jeśli mają eksportery Prometheusa).
- Służy jako źródło danych dla Grafany, gdzie faktycznie oglądasz stan homelabu.
  
Szczegółowe dashboardy i konfigurację źródła danych opisuję w sekcji:
`Usługi → Grafana`.