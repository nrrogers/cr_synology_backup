# cr_synology_backup

Ansible role to deploy a restic-over-SFTP backup wrapper script to a Synology NAS, targeting the harvest TrueNAS box. Used by proteus and bounty.

The wrapper script handles:

- healthcheck.io start / success / failure pings (with log body posted on success and failure)
- Local log capture and rotation (default: 30-day retention)
- restic retention via `restic forget --prune`
- `trap ERR` so any failure mid-script fires the failure ping

Designed to live inside an encrypted DSM shared folder (e.g. `/volume1/docker/scripts/`) so the SSH key, repo password file, wrapper script, and logs are all unreachable until the folder is manually unlocked after a reboot. The healthcheck.io endpoint should be configured with a cron schedule and grace period so a missed run after a forgotten unlock raises the missing-run alert.

## What this role does

- Renders and deploys the wrapper script from a Jinja template
- Deploys the SSH private key (from a vaulted variable)
- Deploys the restic repo password file (from a vaulted variable)
- Ensures the base directory and log directory exist with safe permissions
- Asserts that `restic` is installed on the target before doing anything else

## What this role does NOT do

- **Install restic.** DSM doesn't expose Package Center via a stable Ansible-friendly API. Install restic manually from SynoCommunity, or pre-bake it.
- **Create the DSM Task Scheduler job.** Same reason. Configure the scheduled task in the DSM UI (root user, daily off-hours, command = `{{ restic_synology_script_path }}` rendered to its absolute path).
- **Run `restic init`.** This is a one-time manual step per host, before the first scheduled run.

## Required variables

These have no defaults and must be set per host (typically in `host_vars/<host>.yml`):

| Variable | Purpose |
| --- | --- |
| `restic_synology_repo_url` | Full restic SFTP URL including the per-host subdir |
| `restic_synology_ssh_target_host` | IP or hostname for the SFTP connection |
| `restic_synology_repo_password` | restic repo password — **vault me** |
| `restic_synology_ssh_private_key` | SSH private key contents — **vault me** |
| `restic_synology_healthcheck_uuid` | healthcheck.io UUID for this host's endpoint — **vault me** |
| `restic_synology_backup_paths` | List of absolute paths to back up |

## Variables with defaults

See `defaults/main.yml` for the full list. Notable ones:

| Variable | Default |
| --- | --- |
| `restic_synology_base_dir` | `/volume1/docker/scripts` |
| `restic_synology_ssh_user` | `svc_restic` |
| `restic_synology_healthcheck_base_url` | `https://hc-ping.com` |
| `restic_synology_retention.keep_daily` | `7` |
| `restic_synology_retention.keep_weekly` | `4` |
| `restic_synology_retention.keep_monthly` | `12` |
| `restic_synology_retention.keep_yearly` | `2` |
| `restic_synology_log_retention_days` | `30` |
| `restic_synology_excludes` | DSM cruft + the password file |

## Example host_vars

```yaml
restic_synology_repo_url: "sftp:svc_restic@192.168.77.89:/mnt/harvest/backups/backups_restic/proteus"
restic_synology_ssh_target_host: "192.168.77.89"
restic_synology_repo_password: "{{ vault_proteus_restic_repo_password }}"
restic_synology_ssh_private_key: "{{ vault_restic_backup_id_ed25519_private }}"
restic_synology_healthcheck_uuid: "{{ vault_proteus_restic_healthcheck_uuid }}"
restic_synology_backup_paths:
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
