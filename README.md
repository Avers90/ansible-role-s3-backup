# ansible-role-s3-backup

Ansible role that installs [rclone](https://rclone.org/) and configures an encrypted daily backup
to **any S3-compatible storage** (AWS S3, Beget, Hetzner, Selectel, Wasabi, Scaleway, MinIO, etc.)
for:

- **WireGuard** — interface config and keys
- **WGDashboard** — SQLite databases and INI configs
- **AdGuard Home** — YAML config and stats database

Backups are encrypted with `rclone crypt` (AES-256) before upload. Plain-text passwords are obscured
automatically by the role — no manual `rclone obscure` required.

## Requirements

- Debian or Ubuntu
- The services to be backed up must already be installed
- An S3 bucket with access credentials on any S3-compatible provider

## Variables

```yaml
## rclone version ("latest" or pinned e.g. "v1.68.2")
s3_backup_rclone_version: "latest"

## rclone remote names inside /etc/rclone/rclone.conf
s3_backup_rclone_remote_name: "beget-s3"
s3_backup_rclone_crypt_name:  "beget-s3-crypt"

## S3 provider for rclone: AWS | Other | Minio | Wasabi | Scaleway | DigitalOcean | ...
## "Other" covers any S3-compatible endpoint (Beget, Hetzner, Selectel, Yandex, ...).
s3_backup_s3_provider: "Other"

## Endpoint (leave empty for AWS — region alone is enough)
s3_backup_s3_endpoint: "s3.ru1.storage.beget.cloud"
s3_backup_s3_region:   "ru-1"

## Required for MinIO and some self-hosted S3 gateways
s3_backup_s3_force_path_style: false

## S3 credentials
s3_backup_s3_bucket: ""         # REQUIRED
s3_backup_s3_access_key: ""     # REQUIRED
s3_backup_s3_secret_key: ""     # REQUIRED

## Encryption (plain text - obscured automatically)
s3_backup_crypt_password: ""    # REQUIRED - generate: openssl rand -base64 32
s3_backup_crypt_password2: ""   # REQUIRED - salt, generate separately

## Retention and schedule
s3_backup_retention_days: 7
s3_backup_cron_hour: "3"
s3_backup_cron_minute: "0"

## Enable/disable per service
s3_backup_wireguard_enabled: true
s3_backup_wgdashboard_enabled: true
s3_backup_adguardhome_enabled: true

## AdGuard Home exclusions
s3_backup_adguardhome_exclude_filters: true   # filter lists (large, re-downloaded on startup)
s3_backup_adguardhome_exclude_querylog: true  # query log (large, not critical for restore)
```

All variables are documented in `defaults/main.yml`.

## Provider examples

### AWS S3

```yaml
s3_backup_rclone_remote_name: "aws-s3"
s3_backup_rclone_crypt_name:  "aws-s3-crypt"
s3_backup_s3_provider: "AWS"
s3_backup_s3_endpoint: ""                # not required for AWS
s3_backup_s3_region:   "eu-central-1"
```

### Beget (default)

```yaml
s3_backup_s3_provider: "Other"
s3_backup_s3_endpoint: "s3.ru1.storage.beget.cloud"
s3_backup_s3_region:   "ru-1"
```

### Hetzner Object Storage

```yaml
s3_backup_rclone_remote_name: "hetzner-s3"
s3_backup_rclone_crypt_name:  "hetzner-s3-crypt"
s3_backup_s3_provider: "Other"
s3_backup_s3_endpoint: "fsn1.your-objectstorage.com"   # or nbg1 / hel1
s3_backup_s3_region:   "fsn1"
```

### Selectel

```yaml
s3_backup_rclone_remote_name: "selectel-s3"
s3_backup_rclone_crypt_name:  "selectel-s3-crypt"
s3_backup_s3_provider: "Other"
s3_backup_s3_endpoint: "s3.storage.selcloud.ru"
s3_backup_s3_region:   "ru-1"
```

### MinIO / self-hosted

```yaml
s3_backup_rclone_remote_name: "minio"
s3_backup_rclone_crypt_name:  "minio-crypt"
s3_backup_s3_provider: "Minio"
s3_backup_s3_endpoint: "minio.example.com:9000"
s3_backup_s3_region:   "us-east-1"
s3_backup_s3_force_path_style: true
```

### Wasabi

```yaml
s3_backup_rclone_remote_name: "wasabi-s3"
s3_backup_rclone_crypt_name:  "wasabi-s3-crypt"
s3_backup_s3_provider: "Wasabi"
s3_backup_s3_endpoint: "s3.eu-central-1.wasabisys.com"
s3_backup_s3_region:   "eu-central-1"
```

## Usage

```yaml
- hosts: vpn
  roles:
    - role: ansible-role-wireguard
    - role: ansible-role-wgdashboard
    - role: ansible-role-adguardhome
    - role: ansible-role-s3-backup
      vars:
        s3_backup_s3_bucket: "your-bucket-name"
        s3_backup_s3_access_key: "your-access-key"
        s3_backup_s3_secret_key: "your-secret-key"
        s3_backup_crypt_password: "strong-random-password"
        s3_backup_crypt_password2: "another-random-salt"
```

Store credentials in `group_vars` encrypted with `ansible-vault`.

## S3 Structure

After decryption the bucket contains:

```
<hostname>/
├── wireguard/
│   └── 2026-02-27_030000.tar.gz
├── wgdashboard/
│   └── 2026-02-27_030000.tar.gz
└── adguardhome/
    └── 2026-02-27_030000.tar.gz
```

In S3 the filenames and directory names are encrypted (rclone crypt `filename_encryption = standard`).

## What is backed up

| Service | Files |
|---------|-------|
| WireGuard | `/etc/wireguard/<interface>.conf`, `*_privatekey`, `*_publickey` |
| WGDashboard | `src/db/*.db`, `src/wg-dashboard.ini`, `src/ssl-tls.ini` |
| AdGuard Home | `AdGuardHome.yaml`, `data/stats.db` |

AdGuard Home exclusions (controlled by variables):
- `data/filters/` — **always excluded**, large, re-downloaded automatically on startup
- `data/querylog.json*` — **excluded by default** (`s3_backup_adguardhome_exclude_querylog: true`), not critical for restore, can be large (10–100 MB/day depending on traffic)

## Manual run

```bash
/usr/local/sbin/s3-backup.sh
```

Logs: `/var/log/s3-backup.log`

## Restore

Substitute `<crypt-remote>` with whatever you set in `s3_backup_rclone_crypt_name`
(default: `beget-s3-crypt`):

```bash
# Install rclone and configure /etc/rclone/rclone.conf, then:
rclone --config /etc/rclone/rclone.conf copy <crypt-remote>:<hostname>/wireguard/ ./restore/wireguard/
tar -xzf ./restore/wireguard/<archive>.tar.gz -C /etc/wireguard/
```

## Security

| Measure | Details |
|---------|---------|
| `rclone.conf` permissions | `0600`, owner `root` |
| Config directory | `0700`, owner `root` |
| Backup script | `0700`, owner `root` |
| Staging directory | `0700`, removed via `trap EXIT` after each run |
| Archive files | `0600` before upload, removed after |
| Ansible tasks with secrets | `no_log: true` |
| Password obscuring | Via `rclone obscure -` (stdin, not argument — not visible in `ps aux`) |
