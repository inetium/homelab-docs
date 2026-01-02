# DNS – Pi-hole + MikroTik

## Rola DNS w homelabie

W moim homelabie DNS jest rozwiązany w oparciu o:

- **Pi-hole** uruchomiony w Dockerze na Raspberry Pi 5 (`10.10.20.10`) – filtr reklam + lokalne rekordy DNS.
- **Routery MikroTik (MT1 / MT2 / MT3)** – rozdają adresy IP klientom (DHCP) i wskazują Pi-hole jako główny serwer DNS.

Dzięki temu:

- cały ruch DNS z sieci LAN przechodzi przez Pi-hole,
- mam kontrolę nad lokalnymi nazwami `*.lab` (np. `portainer.lab`, `gitea.lab`, `docs.lab`),
- mogę centralnie filtrować reklamy i złośliwe domeny.

---

## Pi-hole na Raspberry Pi 5

### Lokalizacja i podstawowe założenia

- Serwer: **Raspberry Pi 5**
- Adres IP: `10.10.20.10` (w głównej podsieci `10.10.20.0/24`)
- Uruchomienie: w kontenerze Dockera (na rpi5)
- Funkcje:
  - główny serwer DNS dla klientów LAN,
  - filtr reklam / trackerów,
  - serwer lokalnych rekordów typu A dla domen `*.lab`.

### Dostęp do panelu Pi-hole

Panel WWW Pi-hole jest dostępny z LAN (np. przez IP lub za Traefikiem – zależnie od konfiguracji):

- Adres IP bezpośrednio:  
  `http://10.10.20.10/admin`
- Logowanie hasłem ustawionym podczas konfiguracji Pi-hole.

---

## MikroTik – kierowanie DNS do Pi-hole

### DHCP – rozdawanie adresów klientom

W sieci LAN za adresację odpowiadają routery MikroTik (np. MT1 jako główny).  
W konfiguracji DHCP Mikrotika ustawiam:

- **DNS Server**: `10.10.20.10` (adres Pi-hole),
- opcjonalnie drugi serwer DNS (np. publiczny) – zazwyczaj **nie ustawiam**, żeby wymusić korzystanie z Pi-hole.

Przykładowa logika (uogólniona):

- Klient (PC/telefon) dostaje z DHCP:
  - IP z puli `10.10.20.0/24`,
  - bramę (gateway) – np. IP MT1,
  - **DNS = 10.10.20.10** (Pi-hole).
- Klient wysyła zapytania DNS na `10.10.20.10:53`.
- Pi-hole:
  - najpierw sprawdza swoje listy blokowania,
  - rozwiązuje nazwy lokalne `*.lab`,
  - pozostałe domeny przekazuje dalej do upstream DNS (np. Cloudflare/Google/ISP).

---

## Lokalne domeny *.lab

W Pi-hole mam dodane lokalne rekordy typu A dla podstawowych usług homelabu:

- `portainer.lab` → `10.10.20.10`
- `gitea.lab` → `10.10.20.10`
- `docs.lab` → `10.10.20.10`
- (docelowo również:)
  - `grafana.lab` → `10.10.20.10`
  - `kuma.lab` → `10.10.20.10`
  - `loki.lab` → `10.10.20.10`
  - `traefik.lab` → `10.10.20.10`

Dzięki temu:

- z dowolnego komputera w LAN mogę wejść na usługi po „ludzkich” nazwach:
  - `http://portainer.lab`
  - `http://gitea.lab`
  - `http://docs.lab`
- Traefik na rpi5, opierając się na nagłówku `Host`, potrafi przekierować ruch HTTP/HTTPS do właściwych kontenerów.

---

## Testowanie rozwiązywania nazw

Do szybkiej weryfikacji poprawnego działania DNS w LAN używam poleceń typu:


`nslookup portainer.lab`
`nslookup gitea.lab`
`nslookup docs.lab` 
Oczekiwane odpowiedzi:

- serwer DNS: `10.10.20.10`
- adresy:
  - `portainer.lab` → `10.10.20.10`
  - `gitea.lab` → `10.10.20.10`
  - `docs.lab` → `10.10.20.10`

Jeśli odpowiedzi są inne (np. zewnętrzny DNS), oznacza to, że:

- klient nie korzysta z Pi-hole (ma inny DNS),
- lub DHCP na MikroTiku nie został poprawnie zaktualizowany.

---

## Integracja z Traefikiem

Pi-hole zapewnia **mapowanie nazw `*.lab` na IP rpi5**, a Traefik zajmuje się rozdzielaniem ruchu HTTP/HTTPS do kontenerów.

Przykładowy przepływ dla `http://gitea.lab`:

1. Klient wysyła zapytanie DNS: `gitea.lab` → pyta Pi-hole (`10.10.20.10`).
2. Pi-hole odpowiada: `gitea.lab = 10.10.20.10`.
3. Przeglądarka łączy się z `10.10.20.10:80` z nagłówkiem `Host: gitea.lab`.
4. Traefik na rpi5:
   - ma router z regułą `Host("gitea.lab")`,
   - przekierowuje ruch do kontenera Gitea na porcie 3000.
5. Użytkownik widzi interfejs Gitea.

Analogicznie działa to dla:

- `portainer.lab`
- `docs.lab`
- `grafana.lab`
- i innych usług wystawionych za Traefikiem.

---

## Podsumowanie

- **Pi-hole na rpi5 (`10.10.20.10`):**
  - centralny DNS,
  - filtr reklam,
  - lokalne domeny `*.lab`.

- **MikroTik (MT1/MT2/MT3):**
  - rozdają adresy IP klientom,
  - wskazują Pi-hole jako DNS (`10.10.20.10`).

- **Traefik:**
  - używa nazw `*.lab` z DNS do kierowania ruchu HTTP/HTTPS na konkretne usługi.

W efekcie cały DNS w homelabie jest scentralizowany, łatwy do monitorowania i rozbudowy (nowe usługi → nowe rekordy `*.lab` w Pi-hole + odpowiednie reguły w Traefiku).