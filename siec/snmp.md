# HOMELAB – schemat wdrożenia SNMP w labie

Ten dokument opisuje krok po kroku **wdrożenie SNMP** w labie na MikroTikach (MT1/MT3) oraz integrację z `snmp_exporter` i Prometheusem działającymi na Raspberry Pi 5 (RPi5).

Celem jest:
- zbieranie metryk SNMP z MT1 (i docelowo MT3),
- ekspozycja ich przez `snmp_exporter`,
- scraping w Prometheusie,
- dalsza wizualizacja w Grafanie.

---

## 1. Założenia i adresy

### 1.1. Sieć

- VLAN dla serwerów laba: **VLAN20 – LAB-SRV**
  - Sieć: `10.10.20.0/24`
  - Gateway: `10.10.20.1` (MT1)
- RPi5:
  - VLAN20, IP z DHCP (założenie): `10.10.20.10`
- MT1 (lab core router):
  - Interfejs VLAN20: `10.10.20.1/24`
- MT3 (lab switch/AP):
  - Interfejs VLAN20: `10.10.20.x/24` (dowolny adres z tej sieci, ustalony na MT3)

### 1.2. Rola urządzeń

- **MT1** – główny router laba, terminacja VLANów, gateway dla VLAN20.
- **MT3** – access-switch + AP WiFi, wpięty trunkowo do MT1 (VLAN10/20/40).
- **RPi5** – serwer monitoringu:
  - `snmp_exporter` (9116/tcp),
  - Prometheus (9090/tcp),
  - Grafana (3000/tcp).

---

## 2. Konfiguracja SNMP na MikroTiku (MT1 / MT3)

Poniższe kroki wykonujemy na każdym MikroTiku, który chcemy monitorować (MT1, później MT3).

### 2.1. Włączenie SNMP

CLI (RouterOS):

```rsc
/snmp set enabled=yes
```
Opcjonalnie możesz ustawić podstawowe informacje (nie wymagane do działania, ale przydatne w monitoringu):

```rsc
Copy
/snmp set contact="admin@lab" location="HOMELAB-RACK"
```
### 2.2. Community dla monitoringu
Na potrzeby monitoringu używamy SNMPv2c z community public.
Możliwe są dwa warianty bezpieczeństwa:

**Wariant A – szybki (pełny dostęp z labu)**
Community public dostępne z całej podsieci VLAN20 (`10.10.20.0/24`):

```rsc
/snmp community
add name=public addresses=10.10.20.0/24 security=none
```
**Wariant B – bardziej restrykcyjny (tylko RPi5)**
Community public-lab dostępne wyłącznie z adresu RPi5 (`10.10.20.10`):

```rsc
/snmp community
add name=public-lab addresses=10.10.20.10/32 security=none
```
>W konfiguracji Prometheusa i `snmp_exporter` przyjęliśmy użycie `public`.  
>Jeśli chcesz, możesz później przejść na `public-lab` – wymaga to tylko zmiany community w `auth` w `snmp.yml`.

### 2.3. Firewall na MikroTiku (opcjonalnie, ale zalecane)
Jeżeli masz domyślnie restrykcyjne reguły firewall, dodaj regułę akceptującą SNMP z VLAN20:

```rsc
/ip firewall filter
add chain=input action=accept protocol=udp dst-port=161 \
    src-address=10.10.20.0/24 comment="Allow SNMP from LAB-SRV (VLAN20)"
```
Na MT3 – analogicznie, jeśli ma firewall na input.

## 3. Test SNMP z RPi5
Na RPi5 zainstaluj klienta SNMP:

```bash
sudo apt update
sudo apt install -y snmp
```
Przetestuj SNMP do MT1:

```bash
snmpwalk -v2c -c public 10.10.20.1
```
Jeśli widzisz odpowiedzi (`iso.3.6.1.2.1...`), SNMP działa.  
Analogicznie dla MT3 (po włączeniu SNMP):

```bash
snmpwalk -v2c -c public <IP_MT3_VLAN20>
```
## 4. `snmp_exporter` na RPi5
### 4.1. Struktura katalogów
Na RPi5 (użytkownik `marcin`):

```text
~/monitoring
├── docker-compose.yml
└── snmp_exporter
    └── snmp.yml
```
### 4.2. Pozyskanie referencyjnego snmp.yml
Używamy obrazu `prom/snmp-exporter:latest` (v0.29.0, format „auth-split”).

```bash
cd ~/monitoring

# Kontener tymczasowy
docker run --name snmp_exporter_debug -d prom/snmp-exporter:latest

# Skopiowanie domyślnego pliku konfiguracyjnego
docker cp snmp_exporter_debug:/etc/snmp_exporter/snmp.yml snmp_exporter/snmp.yml

# Usunięcie kontenera tymczasowego
docker rm -f snmp_exporter_debug
```
Plik `snmp_exporter/snmp.yml` zawiera m.in.:

- `auths`: – różne definicje uwierzytelnienia, w tym `public_v2`:
```yaml
auths:
  public_v2:
    community: public
    version: 2
    # ...
```
- `modules`: – w tym moduł `if_mib` (standardowy MIB interfejsów):
```yaml
modules:
  if_mib:
    walk:
      - 1.3.6.1.2.1.2      # interfaces
      - 1.3.6.1.2.1.31.1.1 # ifXTable
    # ...
```
Nie trzeba zmieniać struktury – wystarczy użyć istniejącego modułu `if_mib` i `auth public_v2`.

### 4.3. Konfiguracja usługi snmp_exporter (Docker Compose)
Fragment `docker-compose.yml`:

```yaml
services:
  snmp_exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp_exporter
    restart: unless-stopped
    volumes:
      - ./snmp_exporter/snmp.yml:/etc/snmp_exporter/snmp.yml:ro
    command:
      - "--config.file=/etc/snmp_exporter/snmp.yml"
    ports:
      - "9116:9116"
    networks:
      - mon_net
``
Start usługi:

```bash
cd ~/monitoring
docker compose up -d snmp_exporter
docker compose logs snmp_exporter | tail
```
Sprawdzenie, czy exporter działa:

```bash
curl "http://localhost:9116/metrics" | head
```
Test SNMP przez exporter do MT1:

```bash
curl "http://localhost:9116/snmp?target=10.10.20.1&module=if_mib&auth=public_v2" | head
```
Oczekiwany wynik (przykład):

```text
# HELP ifAdminStatus The desired state of the interface - 1.3.6.1.2.1.2.2.1.7
# TYPE ifAdminStatus gauge
ifAdminStatus{ifDescr="bridge-lab",ifName="bridge-lab",...} 1
ifAdminStatus{ifDescr="ether1",ifName="ether1",...} 1
...
```
## 5. Integracja z Prometheusem
### 5.1. Struktura
```text
~/monitoring
├── docker-compose.yml
└── prometheus
    └── prometheus.yml
```
W `docker-compose.yml` usługa Prometheus:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    networks:
      - mon_net
```
### 5.2. Konfiguracja joba SNMP w prometheus.yml
Przykładowa minimalna konfiguracja:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Self-scrape Prometheusa
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  # SNMP – MikroTiki
  - job_name: "snmp_mikrotik"
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [public_v2]
    static_configs:
      - targets:
          - "10.10.20.1"   # MT1
          # - "<IP_MT3_VLAN20>"   # MT3 – po włączeniu SNMP
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: "snmp_exporter:9116"
```
Restart Prometheusa:

```bash
cd ~/monitoring
docker compose restart prometheus
```
Sprawdzenie statusu targetów:

 - `http://10.10.20.10:9090/targets`
Oczekiwany stan:

 - job `snmp_mikrotik → UP`
- target `10.10.20.1` (oraz IP MT3, jeśli dodany) → `UP`
## 6. Podstawowe zapytania PromQL (w kontekście SNMP)
Kilka przydatnych zapytań opartych o if_mib:

**Sumaryczny ruch IN (bps) per router:**

```promql
sum by (instance) (rate(ifHCInOctets{job="snmp_mikrotik"}[$__rate_interval])) * 8
```
Sumaryczny ruch OUT (bps) per router:

```promql
sum by (instance) (rate(ifHCOutOctets{job="snmp_mikrotik"}[$__rate_interval])) * 8
```
Ruch na konkretnym porcie (np. ether2 na MT1):

```promql
rate(ifHCInOctets{job="snmp_mikrotik", instance="10.10.20.1", ifName="ether2"}[5m]) * 8
```
Top 10 interfejsów wg ruchu IN:

```promql

topk(10, rate(ifHCInOctets{job="snmp_mikrotik"}[5m]) * 8)
```
> ### Uwaga o ostrzeżeniu PromQL:  
> **Prometheus może pokazywać:**
> `metric might not be a counter, name does not end in _total/_sum/_count/_bucket: "ifHCInOctets"`
Metryki SNMP `ifHCInOctets / ifHCOutOctets` są licznikami rosnącymi, ale ich nazwy nie trzymają się konwencji Prometheusa (`_total`).
Użycie `rate(ifHCInOctets[5m])` / `rate(ifHCOutOctets[5m])` jest poprawne – to tylko informacja, którą można zignorować.

## 7. Szybka checklista wdrożenia SNMP w labie
1. MT1 / MT3

 - `/snmp set enabled=yes`
 - Utworzone community `public` (lub `public-lab` z adresem `10.10.20.10/32`).
 - Reguła firewall dopuszczająca UDP/161 z `10.10.20.0/24` (jeśli firewall jest restrykcyjny).
2. RPi5
 - `snmpwalk -v2c -c public 10.10.20.1` działa.
 - `snmp_exporter` uruchomiony (`docker compose up -d snmp_exporter`).
 - `curl "http://localhost:9116/snmp?target=10.10.20.1&module=if_mib&auth=public_v2"` zwraca metryki.
 - Prometheus skonfigurowany z jobem `snmp_mikrotik`.
 - W `http://10.10.20.10:9090/targets` target `10.10.20.1` jest `UP`.
3. Docelowo
 - MT3 dodany jako kolejny target w jobie snmp_mikrotik.
 - Dashboard w Grafanie pokazuje ruch i status interfejsów na MT1/MT3.