# cr_synology_backup

Ansible role to deploy a restic-over-SFTP backup wrapper script to a Synology NAS, targeting the harvest TrueNAS box. Used by proteus and bounty.

The wrapper script handles:

- healthcheck.io start / success / failure pings, with short status messages in the body (`Backup successful for location: proteus`, etc.) — matches the autorestic convention used by the Docker fleet
- Local log capture and rotation (default: 30-day retention)
- restic retention via `restic forget --prune`
- `trap ERR` so any failure mid-script fires the failure ping

Designed to live inside an encrypted DSM shared folder (e.g. `/volume1/docker/scripts/`) so the SSH key, repo password file, wrapper script, and logs are all unreachable until the folder is manually unlocked after a reboot. The healthcheck.io endpoint should be configured with a cron schedule and grace period so a missed run after a forgotten unlock raises the missing-run alert.

## What this role does

- Renders and deploys the wrapper script from a Jinja template
- Deploys the SSH private key (from a vaulted variable)
- Deploys the restic repo password file (from a vaulted variable)
- Ensures the base directory and log directory exist with safe permissions
- Asserts that `restic` is installed at the configured path before doing anything else

## What this role does NOT do

- **Install restic.** DSM doesn't expose Package Center via a stable Ansible-friendly API. Install restic manually from SynoCommunity, or pre-bake it.
- **Create the DSM Task Scheduler job.** Same reason. Configure the scheduled task in the DSM UI (root user, daily off-hours, command = the absolute path to the rendered wrapper script).
- **Run `restic init`.** This is a one-time manual step per host, before the first scheduled run.

## Required variables

These have no defaults and must be set per host (typically in `host_vars/<host>.yml`):

| Variable | Purpose |
| --- | --- |
| `cr_synology_backup_repo_url` | Full restic SFTP URL including the per-host subdir |
| `cr_synology_backup_ssh_target_host` | IP or hostname for the SFTP connection |
| `cr_synology_backup_repo_password` | restic repo password — **vault me** |
| `cr_synology_backup_ssh_private_key` | SSH private key contents — **vault me** |
| `cr_synology_backup_healthcheck_url` | Full healthcheck.io ping URL incl. UUID — **vault me** |
| `cr_synology_backup_paths` | List of absolute paths to back up |

## Variables with defaults

See `defaults/main.yml` for the full list. Notable ones:

| Variable | Default |
| --- | --- |
| `cr_synology_backup_base_dir` | `/volume1/docker/scripts` |
| `cr_synology_backup_restic_binary` | `/usr/local/bin/restic` |
| `cr_synology_backup_ssh_user` | `svc_restic` |
| `cr_synology_backup_location_name` | `{{ inventory_hostname }}` |
| `cr_synology_backup_retention.keep_last` | `2` |
| `cr_synology_backup_retention.keep_daily` | `7` |
| `cr_synology_backup_retention.keep_weekly` | `4` |
| `cr_synology_backup_retention.keep_monthly` | `12` |
| `cr_synology_backup_retention.keep_yearly` | `2` |
| `cr_synology_backup_log_retention_days` | `30` |
| `cr_synology_backup_excludes` | DSM cruft + the password file |

## Example host_vars

```yaml
cr_synology_backup_repo_url: "sftp:svc_restic@192.168.77.89:/mnt/harvest/backups/backups_restic/proteus"
cr_synology_backup_ssh_target_host: "192.168.77.89"
cr_synology_backup_repo_password: "{{ vault_proteus_restic_repo_password }}"
cr_synology_backup_ssh_private_key: "{{ vault_restic_backup_id_ed25519_private }}"
cr_synology_backup_healthcheck_url: "{{ vault_proteus_restic_healthcheck_url }}"
cr_synology_backup_paths:
  - /volume1/photo
  - /volume1/video
  - /volume1/music
  - /volume1/documents
  - /volume1/homes
```

## Example play

```yaml
- hosts: synology_restic_clients
  roles:
    - cr_synology_backup
```

## See also

- `truenas-migration-plan-revised.md` in the homelab Obsidian vault — overall migration plan
- `Restic Synology Backups to TrueNAS via SFTP.md` — full deployment / DR notes
