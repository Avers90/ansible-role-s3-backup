# ansible-role-s3-backup

Ansible role that installs [rclone](https://rclone.org/) and configures an encrypted daily backup to S3 (Beget Object Storage) for:

- **WireGuard** — interface config and keys
- **WGDashboard** — SQLite databases and INI configs
- **AdGuard Home** — YAML config, stats and query log

Backups are encrypted with `rclone crypt` (AES-256) before upload. Plain-text passwords are obscured automatically by the role — no manual `rclone obscure` required.

## Requirements

- Debian or Ubuntu
- The services to be backed up must already be installed
- A Beget S3 bucket with access credentials

## Variables

```yaml
## rclone version ("latest" or pinned e.g. "v1.68.2")
s3_backup_rclone_version: "latest"

## Beget S3 settings
s3_backup_s3_endpoint: "s3.ru1.storage.beget.cloud"
s3_backup_s3_region: "ru-1"
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
```

All variables are documented in `defaults/main.yml`.

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
| AdGuard Home | `AdGuardHome.yaml`, `data/stats.db`, `data/querylog.json*` |

AdGuard Home filter lists (`data/filters/`) are **excluded** — they are large and re-downloaded automatically on startup.

## Manual run

```bash
/usr/local/sbin/s3-backup.sh
```

Logs: `/var/log/s3-backup.log`

## Restore

```bash
# Install rclone and configure /etc/rclone/rclone.conf, then:
rclone --config /etc/rclone/rclone.conf copy beget-s3-crypt:<hostname>/wireguard/ ./restore/wireguard/
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
