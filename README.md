# dml-ac-db-backup

Automated, zero-build MySQL 8.4 backup container for **AzerothCore** Docker environments.  
Designed as a drop-in sidecar add-on for **Dad's MMO Lab (DML)** AzerothCore setups, and equally ready to run **standalone** alongside any existing AzerothCore Docker stack.

## Features
- **Zero custom image build:** Uses the official `mysql:8.4` image directly.
- **Consistent live snapshots:** Uses `--single-transaction --routines --triggers` and compresses on-the-fly with `gzip` (`ac_backup_YYYYMMDD_HHMMSS_timer.sql.gz`).
- **Atomic writes:** Writes to a hidden `.tmp` file first and renames only after `mysqldump` and `gzip` succeed (`set -o pipefail`), preventing half-written or empty archives on disk/power errors.
- **Express shutdown backup (`SIGTERM`):** Traps `docker compose down` / `docker stop` (`stop_grace_period: 30s`) and immediately creates a final `*_stop.sql.gz` snapshot before the container exits.
- **Minimum retention guard (`MIN_KEEP`):** Deletes backups older than `RETENTION_DAYS`, **but always preserves at least the newest `MIN_KEEP` backups** (default: `10`) even if your server was offline for weeks.

---

## Option A: Sidecar in Dad's MMO Lab (DML) AzerothCore Setup
1. Copy `docker-compose.override.yml` into your AzerothCore server directory (or merge the `ac-database` and `ac-db-backup` blocks into your existing `docker-compose.override.yml`).
2. Optionally copy variables from `.env.example` into your `.env` file to customize databases, interval, or retention.
3. Start or update your stack:
   ```bash
   docker compose up -d
   ```
Backups will be saved to `./sql_scripts/backups/`.

---

## Option B: Standalone Alongside an Existing AzerothCore Stack
1. Clone this repository into a separate folder:
   ```bash
   git clone https://github.com/psylopterix/dml-ac-db-backup.git
   cd dml-ac-db-backup
   cp .env.example .env
   ```
2. Find the Docker network name of your running AzerothCore stack:
   ```bash
   docker network ls
   ```
   Look for the network ending in `_ac-network` (e.g. `wow-server-playerbots_ac-network`).
3. Create the local backup directory (so your user owns the folder instead of `root`) and configure `.env` (`AC_DOCKER_NETWORK`, `BACKUP_HOST_DIR`, `DOCKER_DB_ROOT_PASSWORD`):
   ```bash
   mkdir -p backups
   ```
4. Start the backup service:
   ```bash
   docker compose up -d
   ```
Backups will be saved to `BACKUP_HOST_DIR` (default: `./backups/`).

---

## Configuration

| Variable | Default | Description |
| :--- | :--- | :--- |
| `TZ` | `Europe/Berlin` | Container timezone for backup timestamps |
| `AC_DOCKER_NETWORK` | `wow-server-playerbots_ac-network` | External Docker network name (only relevant for standalone mode) |
| `BACKUP_HOST_DIR` | `./backups` | Host directory for backup files (only relevant for standalone mode) |
| `DOCKER_DB_HOST` | `ac-database` | MySQL hostname / container name |
| `DOCKER_DB_USER` | `root` | MySQL user with read/dump privileges |
| `DOCKER_DB_ROOT_PASSWORD` | `password` | MySQL password |
| `DOCKER_DB_BACKUP_DATABASES` | `acore_auth acore_characters acore_playerbots` | Space-separated list of databases to back up |
| `DOCKER_DB_BACKUP_FREQUENCY` | `3600` | Interval between scheduled backups in seconds |
| `DOCKER_DB_RETENTION_DAYS` | `7` | Delete backups older than this many days |
| `DOCKER_DB_BACKUP_MIN_KEEP` | `10` | Minimum number of newest backups to always keep |

---

## Restoring a Backup
To restore any `.sql.gz` archive into a running `ac-database` container:
```bash
gunzip -c backups/ac_backup_YYYYMMDD_HHMMSS_timer.sql.gz | docker exec -i ac-database mysql -uroot -p<YOUR_PASSWORD>
```
