## 0. Wstęp – jak współistnieją IPv4 i IPv6 (dual‑stack)
W Twoim labie nie zastępujemy IPv4 – dokładamy do niego IPv6:

- każde urządzenie ma adres **IPv4 i IPv6** (dual‑stack),
- router (MikroTik) routuje **osobno IPv4 i IPv6**,
- DNS (Pi‑hole) dla tych samych nazw (`*.lab`) zwraca:
    - rekord `A` → IPv4 (`10.10.20.10`),
    - rekord `AAAA` → IPv6 (np. `fd10:10:20::10`),
- klient zwykle **najpierw próbuje IPv6**, a jeśli nie działa – fallback na IPv4.
  
Na start zrobimy **IPv6 tylko w LAN** (ULA, bez internetu po IPv6). Globalny prefix od ISP i pełne IPv6 w Dockerze/Traefiku możemy dodać jako kolejny etap.

## 1. Plan adresacji IPv6 dla labu
Obecna sytuacja (IPv4):

- LAN: `10.10.20.0/24`
- MikroTik (brama): `10.10.20.1`
- rpi5 (Pi‑hole, Traefik, usługi Docker): `10.10.20.10`
Proponowana IPv6 (ULA, tylko wewnętrznie):

- **prefix LAN ULA**: `fd10:10:20::/64`
- **MikroTik (LAN)**: `fd10:10:20::1/64`
- **rpi5**: `fd10:10:20::10/64`
To odpowiada IPv4 `10.10.20.* → fd10:10:20::*` i jest łatwe do ogarnięcia.

## 2. MikroTik – włączenie IPv6 i podstawowa konfiguracja
### 2.1. Sprawdź, czy IPv6 jest włączone
```routeros
/system package print where name~"ipv6"
```
Jeśli pakiet ipv6 jest disabled → włącz i zrób reboot.

Włącz ogólnie IPv6:

```routeros
/ipv6 settings set disable-ipv6=no
```
### 2.2. Utwórz listy interfejsów WAN i LAN (poprawiona wersja)
Masz tylko wbudowane listy (`all, none, dynamic, static`), więc najpierw tworzymy własne:

```routeros
/interface list add name=WAN comment="Uplink/WAN"
/interface list add name=LAN comment="Internal/LAN"
```
Teraz ustalamy, które interfejsy są WAN i LAN:

```routeros
/interface print
/interface bridge print
```
Załóżmy (zmień, jeśli u Ciebie jest inaczej):

**WAN** = `ether1`
**bridge LAN** = `bridge-lan`
Dodanie do list:

```routeros
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=bridge-lan
```
*(Jeśli Twój bridge nazywa się po prostu `bridge`, to użyj `bridge` zamiast `bridge-lan`)*

### 2.3. Dodaj adres IPv6 na bridge LAN (RA/SLAAC dla hostów)
```routeros
/ipv6 address add address=fd10:10:20::1/64 interface=bridge-lan advertise=yes comment="LAN ULA"
```
- `advertise=yes` → MikroTik wysyła RA, hosty same dostaną adresy `fd10:10:20::xxxx` i bramę `fd10:10:20::1`.  
Na tym etapie klienci w LAN powinni zacząć dostawać IPv6 (SLAAC).

### 2.4. Minimalny, poprawny firewall IPv6 (z użyciem list WAN/LAN)
**Uwaga:** jeśli masz już jakieś reguły IPv6 – wstaw te poniżej wysoko (nad ewentualnymi drop all).

```routeros
/ipv6 firewall filter
add chain=input action=accept connection-state=established,related comment="IPv6: allow established, related"
add chain=input action=accept protocol=icmpv6 comment="IPv6: allow ICMPv6"
add chain=input action=accept in-interface-list=LAN comment="IPv6: allow LAN to router"
add chain=input action=drop in-interface-list=WAN comment="IPv6: drop everything from WAN to router"

add chain=forward action=accept connection-state=established,related comment="IPv6: allow established/related forward"
add chain=forward action=accept protocol=icmpv6 comment="IPv6: allow ICMPv6 forward"
add chain=forward action=drop in-interface-list=WAN connection-state=new comment="IPv6: block new from WAN to LAN"
```
To jest **poprawiona** wersja – korzysta z list interfejsów, które właśnie utworzyłeś, więc nie będzie błędu „input does not match any value of interface-list”.

## 3. rpi5 (Ubuntu 25.10) – statyczny IPv6 przez netplan
Sprawdź nazwę interfejsu:

```bash
ip addr
```
Załóżmy, że to `enp1s0`. Edytuj plik netplan, w którym masz skonfigurowane `10.10.20.10` (np. `/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      addresses:
        - 10.10.20.10/24
        - fd10:10:20::10/64
      routes:
        - to: default
          via: 10.10.20.1
      nameservers:
        addresses:
          - 10.10.20.10
```
Uwagi:

- **Nie dodajemy na sztywno** default route IPv6 – przyjdzie z RA od MikroTika (brama `fd10:10:20::1`).
- DNS zostaje na razie po IPv4 (`10.10.20.10` – Pi‑hole), tak jest najprościej na start.
- 
Zastosuj:

```bash
sudo netplan apply
ip -6 addr show
ip -6 route show
```
Powinieneś zobaczyć:

- `fd10:10:20::10/64` na `enp1s0`,
- dodatkowy adres `fd10:10:20::xxxx/64` (SLAAC – normalne),
- domyślną trasę `default via fd10:10:20::1`.
  
Testy z rpi5:

```bash
ping -6 fd10:10:20::1      # MikroTik
ping -6 fd10:10:20::10     # lokalnie po IPv6
```
## 4. Klienci LAN – weryfikacja IPv6
**Windows**
```powershell
ipconfig
```
Sprawdź:

- IPv6 Address zaczynający się od `fd10:10:20::`,
- Default Gateway IPv6: `fd10:10:20::1`.
  
Test ping:

```powershell
ping -6 fd10:10:20::1      # MikroTik
ping -6 fd10:10:20::10     # rpi5
```

**Linux**
```bash
ip -6 addr show
ping -6 fd10:10:20::1
ping -6 fd10:10:20::10
```
Jeśli to działa – **dual‑stack w LAN (L2/L3) jest skonfigurowany poprawnie**.

## 5. Pi‑hole – dodanie rekordów `AAAA` dla `*.lab`
Klienci nadal pytają Pi‑hole po **IPv4** (`10.10.20.10`), ale Pi‑hole może im już zwracać również IPv6.

Wejdź w panel Pi‑hole:

- `[http://10.10.20.10/admin](http://10.10.20.10/admin)` lub `[http://pihole.lab/admin](http://pihole.lab/admin)`
  
Sekcja:

- **Local DNS → DNS Records**
  
Dodaj (lub uzupełnij istniejące A):

- `portainer.lab` → `fd10:10:20::10`
- `gitea.lab` → `fd10:10:20::10`
- w przyszłości np.:
    - `grafana.lab` → `fd10:10:20::10`
    - `kuma.lab` → `fd10:10:20::10`
    - `loki.lab` → `fd10:10:20::10`
    - `traefik.lab` → `fd10:10:20::10`
Rekordy A (IPv4 = `10.10.20.10`) **zostawiasz** – będziesz mieć równolegle `A + AAAA` dla tych samych nazw.

Test z klienta:

```bash
nslookup portainer.lab 10.10.20.10
```
Oczekiwany efekt:

- `Address: 10.10.20.10` (A),
- `Address: fd10:10:20::10` (AAAA).
## 6. Test nazw i routingu po IPv6
Na kliencie (Linux lub WSL):

```bash
ping -6 portainer.lab
ping -6 gitea.lab
```
Jeżeli odpowiedź przychodzi z `fd10:10:20::10` → **DNS i routing po IPv6 w LAN działają**.

HTTP/HTTPS:

- `http://portainer.lab`, `http://gitea.lab` – na tym etapie mogą wciąż używać **IPv4** do kontenerów (bo Docker domyślnie nie ma włączonego IPv6 NAT).
- To jest OK – cel tej procedury to **stabilny dual‑stack w LAN + poprawny DNS**.
Pełne HTTP po IPv6 do kontenerów zrobimy w oddzielnym etapie (Docker IPv6 / host mode dla Traefika).
## 7. Co dalej (opcjonalne kolejne etapy)
Kiedy powyższe działa (pingi + DNS po IPv6):

  1. **Globalny IPv6 z ISP (DHCPv6‑PD)**  
     - /ipv6 dhcp-client na interfejsie WAN,
     - przydzielenie prefixu z puli na LAN (from-pool=),
     - aktualizacja firewalla IPv6 (WAN z internetu),
     - test z klienta: ping -6 ipv6.google.com.
  2. **Pełne IPv6 dla usług Docker/Traefik**
     - prosty wariant: Traefik/Pi‑hole w network_mode: host → automatycznie nasłuchują na wszystkich adresach rpi5 (IPv4 + IPv6), backendy mogą pozostać IPv4;
     - „czysty” wariant: włączenie IPv6 w Dockerze + sieć proxy z IPv6, Traefik nasłuchuje na v6, kontenery dostają własne adresy IPv6.