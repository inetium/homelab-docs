# HOMELAB – Konfiguracja backupów na NAS (192.168.1.200)

## 1. Architektura backupów

- **NAS** (`192.168.1.200`)
  - Udział backupów: `\\192.168.1.200\homelab\backup`
    - `proxmox/` – backupy VM/LXC z Proxmoxa
    - `mikrotik/` – eksport konfiguracji MikroTika (`.rsc`)
    - `rpi/` – backup RPi:
      - `srv_docker.tgz` – archiwum `/srv/docker`
      - `gitea_db.sql.gz` – dump bazy Postgres Gitea
  - Udział logów: `\\192.168.1.200\homelab\logs`
    - `homelab_backup_report_YYYY-MM-DD_rpi5.log` – dzienny raport backupów

- **Raspberry Pi 5 (Ubuntu 25.10)**
  - Montuje udziały NAS-a (`/mnt/nas_backup`, `/mnt/nas_logs`)
  - Wykonuje:
    - backup MikroTika (SSH → export → `.rsc` na NAS)
    - backup Dockera (`/srv/docker`) + dump Postgresa Gitea
    - dzienny raport backupów

- **Proxmox**
  - Natywne joby VZDump do storage CIFS `nas-backup` (subdir `backup/proxmox`)

- **MikroTik**
  - Użytkownik `backup` + klucz SSH z RPi
  - Codzienny export konfiguracji do pliku `.rsc`

---

## 2. NAS – montowanie udziałów na RPi

### 2.1. Udział backupów

Plik poświadczeń:

```bash
sudo nano /root/.nas-cred
```
```ini
username=<NAS_USER>
password=<NAS_PASSWORD>
```

```bash
sudo chmod 600 /root/.nas-cred
sudo mkdir -p /mnt/nas_backup
sudo nano /etc/fstab
```

Wpis do /etc/fstab:

```fstab
//192.168.1.200/homelab/backup /mnt/nas_backup cifs credentials=/root/.nas-cred,iocharset=utf8,uid=1000,gid=1003,file_mode=0640,dir_mode=0750,nofail 0 0
```

Zastosowanie:

```bash
sudo mount -a
sudo mkdir -p /mnt/nas_backup/mikrotik
sudo mkdir -p /mnt/nas_backup/rpi
sudo mkdir -p /mnt/nas_backup/proxmox    # opcjonalnie, pod Proxmoxa
```
### 2.2. Udział logów

```bash
sudo mkdir -p /mnt/nas_logs
sudo nano /etc/fstab
```

Dopisz:

```fstab
//192.168.1.200/homelab/logs /mnt/nas_logs cifs credentials=/root/.nas-cred,iocharset=utf8,uid=1000,gid=1003,file_mode=0640,dir_mode=0750,nofail 0 0
```

Zastosowanie:

```bash
sudo mount -a
```

## 3. MikroTik – dostęp SSH z RPi

Na RPi (jako marcin):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

Skopiuj zawartość:

```bash
cat ~/.ssh/id_ed25519.pub
# przykład:
# ssh-ed25519 AAAA... marcin@rpi5
```

Na MikroTiku (CLI):

```routeros
/user add name=backup group=full
/file add name=id_ed25519.pub
/file edit id_ed25519.pub
# wklej CAŁĄ linię z id_ed25519.pub, zapisz i wyjdź
/user ssh-keys import user=backup public-key-file=id_ed25519.pub
/file remove id_ed25519.pub
```

Test z RPi:

```bash
ssh backup@10.10.20.1
```

## 4. Skrypt backupu MikroTika (backup_mikrotik.sh)

Ścieżka: /usr/local/sbin/backup_mikrotik.sh

```bash
sudo nano /usr/local/sbin/backup_mikrotik.sh
```

Treść:

```bash
#!/usr/bin/env bash
set -e

ROUTER_IP="10.10.20.1"
ROUTER_USER="backup"

BACKUP_DIR="/mnt/nas_backup/mikrotik"
DATE="$(date +%Y-%m-%d)"

mkdir -p "${BACKUP_DIR}"

# 1) Na MikroTiku: eksport konfiguracji do pliku
ssh "${ROUTER_USER}@${ROUTER_IP}" "/export hide-sensitive file=backup-tmp"

# 2) Ściągnięcie pliku .rsc na NAS (przez RPi)
scp "${ROUTER_USER}@${ROUTER_IP}:backup-tmp.rsc" "${BACKUP_DIR}/mikrotik_${DATE}.rsc"

# 3) Sprzątanie po sobie na MikroTiku
ssh "${ROUTER_USER}@${ROUTER_IP}" "/file remove backup-tmp.rsc"

# 4) Retencja: usuń pliki starsze niż 10 dni
find "${BACKUP_DIR}" -name "mikrotik_*.rsc" -type f -mtime +10 -delete
```
Uprawnienia i test:

```bash
sudo chmod +x /usr/local/sbin/backup_mikrotik.sh
sudo /usr/local/sbin/backup_mikrotik.sh
ls -l /mnt/nas_backup/mikrotik
```

## 5. Skrypt backupu RPi + Gitea Postgres (backup_rpi.sh)

Ścieżka: /usr/local/sbin/backup_rpi.sh

Założenia:

- katalog danych Dockera: `/srv/docker`
- kontener Postgresa: `gitea-db`
- baza: `gitea`
- user: `gitea`
- hasło pobierane z env `POSTGRES_PASSWORD` w kontenerze

```bash
sudo nano /usr/local/sbin/backup_rpi.sh
```

Treść:

```bash
#!/usr/bin/env bash
set -e

BACKUP_ROOT="/mnt/nas_backup/rpi"
DATE="$(date +%Y-%m-%d)"
DEST_DIR="${BACKUP_ROOT}/${DATE}"

mkdir -p "${DEST_DIR}"

echo "=== [$(date '+%Y-%m-%d %H:%M:%S')] RPi backup started ==="

##############################
# 1) Backup bazy Gitea (Postgres)
##############################

PG_CONTAINER="gitea-db"
PG_DB="gitea"
PG_USER="gitea"

if docker ps --format '{{.Names}}' | grep -q "^${PG_CONTAINER}$"; then
  echo " - Found Postgres container: ${PG_CONTAINER}, creating dump..."

  DB_PASS="$(docker exec "${PG_CONTAINER}" printenv POSTGRES_PASSWORD)"

  if [ -z "${DB_PASS}" ]; then
    echo "   WARNING: POSTGRES_PASSWORD not set in container ${PG_CONTAINER}, skipping DB dump."
  else
    DUMP_FILE="${DEST_DIR}/gitea_db.sql.gz"
    if docker exec -e PGPASSWORD="${DB_PASS}" "${PG_CONTAINER}" \
        pg_dump -U "${PG_USER}" "${PG_DB}" | gzip > "${DUMP_FILE}"; then
      echo "   OK: Gitea DB dump created at ${DUMP_FILE}"
    else
      echo "   ERROR: Failed to create Gitea DB dump."
    fi
  fi
else
  echo " - WARNING: Postgres container '${PG_CONTAINER}' not running, skipping DB dump."
fi

##############################
# 2) Backup katalogu /srv/docker
##############################

echo " - Creating archive of /srv/docker..."
TAR_FILE="${DEST_DIR}/srv_docker.tgz"

tar -czf "${TAR_FILE}" /srv/docker --warning=no-file-changed
echo "   OK: /srv/docker archived to ${TAR_FILE}"

##############################
# 3) Retencja
##############################

echo " - Applying retention policy (delete dirs older than 7 days in ${BACKUP_ROOT})..."
find "${BACKUP_ROOT}" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} +

echo "=== [$(date '+%Y-%m-%d %H:%M:%S')] RPi backup finished ==="
```

Uprawnienia i test:

```bash
sudo chmod +x /usr/local/sbin/backup_rpi.sh
sudo /usr/local/sbin/backup_rpi.sh
ls -R /mnt/nas_backup/rpi
```
Struktura:

```text
/mnt/nas_backup/rpi/YYYY-MM-DD/
  gitea_db.sql.gz
  srv_docker.tgz
```

## 6. Skrypt dziennego raportu (homelab_backup_report.sh)

Ścieżka: /usr/local/sbin/homelab_backup_report.sh

Zapisuje raport na NAS:

/mnt/nas_logs/homelab_backup_report_YYYY-MM-DD_rpi5.log

```bash
sudo nano /usr/local/sbin/homelab_backup_report.sh
```
Treść:

```bash
#!/usr/bin/env bash
set -e

NAS_BACKUP_ROOT="/mnt/nas_backup"
NAS_LOG_ROOT="/mnt/nas_logs"
ROUTER_BACKUP_DIR="${NAS_BACKUP_ROOT}/mikrotik"
RPI_BACKUP_DIR="${NAS_BACKUP_ROOT}/rpi"

REPORT_DATE="$(date +%Y-%m-%d)"
REPORT_TIME="$(date '+%Y-%m-%d %H:%M:%S')"
HOSTNAME="$(hostname)"
REPORT_FILE="${NAS_LOG_ROOT}/homelab_backup_report_${REPORT_DATE}_${HOSTNAME}.log"

log() {
  echo "\$1" | tee -a "${REPORT_FILE}"
}

mkdir -p "${NAS_LOG_ROOT}"

log "================= HOMELAB BACKUP REPORT ================="
log "Host: ${HOSTNAME}"
log "Date: ${REPORT_TIME}"
log "NAS backup root: ${NAS_BACKUP_ROOT}"
log "========================================================="
log ""

# 1) MikroTik
log "1) MikroTik backup"

if [ -d "${ROUTER_BACKUP_DIR}" ]; then
  LAST_ROUTER_BACKUP=$(find "${ROUTER_BACKUP_DIR}" -maxdepth 1 -type f -name "mikrotik_*.rsc" -printf "%T@ %p\n" 2>/dev/null | sort -n | tail -1 | awk '{print \$2}')
  if [ -n "${LAST_ROUTER_BACKUP}" ]; then
    LAST_ROUTER_BACKUP_DATE=$(basename "${LAST_ROUTER_BACKUP}" | sed -E 's/^mikrotik_([0-9]{4}-[0-9]{2}-[0-9]{2}).*/\1/')
    log "  OK: found last MikroTik backup: ${LAST_ROUTER_BACKUP} (date: ${LAST_ROUTER_BACKUP_DATE})"
  else
    log "  WARNING: no MikroTik backup files found in ${ROUTER_BACKUP_DIR}"
  fi
else
  log "  ERROR: MikroTik backup directory does not exist: ${ROUTER_BACKUP_DIR}"
fi

log ""

# 2) RPi (Docker)
log "2) RPi (Docker) backup"

LAST_RPI_DIR=""
LAST_RPI_PATH=""

if [ -d "${RPI_BACKUP_DIR}" ]; then
  LAST_RPI_DIR=$(find "${RPI_BACKUP_DIR}" -maxdepth 1 -type d -printf "%P\n" 2>/dev/null | grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' | sort | tail -1)
  if [ -n "${LAST_RPI_DIR}" ]; then
    LAST_RPI_PATH="${RPI_BACKUP_DIR}/${LAST_RPI_DIR}"
    if [ -f "${LAST_RPI_PATH}/srv_docker.tgz" ]; then
      SIZE_HUMAN=$(du -h "${LAST_RPI_PATH}/srv_docker.tgz" | awk '{print \$1}')
      log "  OK: found last RPi Docker backup: ${LAST_RPI_PATH}/srv_docker.tgz (size: ${SIZE_HUMAN})"
    else
      log "  WARNING: latest RPi backup dir ${LAST_RPI_PATH} does not contain srv_docker.tgz"
    fi
  else
    log "  WARNING: no dated directories (YYYY-MM-DD) found in ${RPI_BACKUP_DIR}"
  fi
else
  log "  ERROR: RPi backup directory does not exist: ${RPI_BACKUP_DIR}"
fi

log ""

# 2a) Gitea Postgres dump
log "2a) Gitea Postgres dump (gitea_db.sql.gz)"

if [ -n "${LAST_RPI_PATH}" ] && [ -d "${LAST_RPI_PATH}" ]; then
  GITEA_DUMP_FILE="${LAST_RPI_PATH}/gitea_db.sql.gz"
  if [ -f "${GITEA_DUMP_FILE}" ]; then
    SIZE_BYTES=$(stat -c%s "${GITEA_DUMP_FILE}")
    SIZE_HUMAN=$(du -h "${GITEA_DUMP_FILE}" | awk '{print \$1}')
    if [ "${SIZE_BYTES}" -gt 0 ]; then
      log "  OK: found Gitea DB dump: ${GITEA_DUMP_FILE} (size: ${SIZE_HUMAN}, bytes: ${SIZE_BYTES})"
    else
      log "  WARNING: Gitea DB dump file exists but is empty: ${GITEA_DUMP_FILE}"
    fi
  else
    log "  WARNING: Gitea DB dump not found in latest RPi backup dir: ${LAST_RPI_PATH}"
  fi
else
  log "  INFO: skipped Gitea DB dump check (no valid latest RPi backup dir)."
fi

log ""
log "================= END OF REPORT ========================="
```
Uprawnienia i test:

```bash
sudo chmod +x /usr/local/sbin/homelab_backup_report.sh
sudo /usr/local/sbin/homelab_backup_report.sh
ls -l /mnt/nas_logs
cat /mnt/nas_logs/homelab_backup_report_$(date +%Y-%m-%d)_rpi5.log
```

## 7. Cron – automatyzacja na RPi
Edytuj crontab roota:

```bash
sudo crontab -e
```
Propozycja harmonogramu:

```cron
# 02:00 - backup MikroTika
0 2 * * * /usr/local/sbin/backup_mikrotik.sh >> /var/log/backup_mikrotik.log 2>&1

# 02:15 - backup RPi (Docker + Gitea DB)
15 2 * * * /usr/local/sbin/backup_rpi.sh >> /var/log/backup_rpi.log 2>&1

# 03:30 - dzienny raport backupów
30 3 * * * /usr/local/sbin/homelab_backup_report.sh >> /var/log/homelab_backup_report.log 2>&1
```
## 8. Proxmox – backup VM/LXC na NAS

1. W GUI Proxmoxa:
 - `Datacenter → Storage → Add → CIFS`
 - ID: `nas-backup`
  - Server: `192.168.1.200`
  - Share: `homelab`
  - Subdir: `backup/proxmox`
  - Content: zaznacz `VZDump backup file`
2. Job backupu:
  - `Datacenter → Backup → Add`
  - Schedule: np. 03:00 codziennie
  - Storage: nas-backup
  - Mode: snapshot
  - Compression: zstd
  - Retencja: np. Keep last: 2
  
Backupy Proxmoxa lądują w:

`\\192.168.1.200\homelab\backup\proxmox\...`

## 9. Szybki checklist testu przywracania
- MikroTik: 
  - Plik `.rsc` na NAS (`mikrotik_YYYY-MM-DD.rsc`).
  - Przywracanie: `/import file-name=mikrotik_YYYY-MM-DD.rsc` na nowym/testowym routerze.
- Gitea (Postgres):
  - Dump `gitea_db.sql.gz` w `rpi/YYYY-MM-DD/`.
  - Przywracanie:
    - zatrzymać kontener gitea
    - w kontenerze Postgresa:
      - `DROP DATABASE gitea; CREATE DATABASE gitea;`
      - wgrać dump przez psql z gitea_db.sql.gz.
- Docker (cały RPi):
  - `srv_docker.tgz` w `rpi/YYYY-MM-DD/`.
  - Przywracanie:
    - na nowym systemie: rozpakować `tar z -C / `(żeby odtworzyć `/srv/docker/...`),
    - uruchomić docker compose up -d w odpowiednich katalogach.
  
- Proxmox:
  - Test: przywrócić wybraną VM jako nową (z innym ID) z backupu na `nas-backup` i upewnić się, że startuje.