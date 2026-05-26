# Deprecated, use https://github.com/adainrivers/dune-dedicated-server-manager

# Dune Server Management Service

Server-local automation for a self-hosted Dune: Awakening dedicated server.

The service runs as the `dune` user on the Linux host. It schedules existing operational scripts, stores historical task runs in SQLite, and exposes a localhost-only dashboard for run history and logs.

## What It Runs

- Database backup every 2 hours.
- Steam update check every 15 minutes.
- Pending update apply check every minute.
- Daily restart warning at 04:30 in the configured service timezone.
- Daily restart at 05:00 in the configured service timezone.
- SQLite run history and task logs.
- Web dashboard on `127.0.0.1:8787` by default.

The service does not expose the dashboard publicly. Use an SSH tunnel for access.

## Repository Layout

```text
src/        TypeScript service source
scripts/    Server-side script dependencies installed to /home/dune/.dune/bin
systemd/    systemd unit file
dist/       Generated build output, ignored by Git
```

Script dependencies included in this repo:

```text
scripts/cron-battlegroup-backup
scripts/cron-battlegroup-update-check
scripts/apply-pending-battlegroup-update
scripts/daily-battlegroup-restart-notice
scripts/daily-battlegroup-restart
scripts/send-dune-broadcast
scripts/send-dune-shutdown-broadcast
scripts/lib/dune-service-common.sh
scripts/install-script-deps.sh
```

The broadcast scripts are safe templates. They do not contain the Dune command-auth token. At install time, provide that token through a private server-local file or environment variable.

## Requirements

On the server:

- Ubuntu host with the Dune BattleGroup already installed.
- `dune` user with working `kubectl`.
- `/home/dune/.dune/bin/battlegroup`.
- SteamCMD at `/home/dune/.local/bin/steamcmd`.
- `bash`, `flock`, `python3`, `curl`, `sudo`, `kubectl`.
- Node.js 24 or newer recommended.

Node must support `node:sqlite`. The tested runtime was:

```bash
node --version
npm --version
```

Expected shape:

```text
v24.x
11.x
```

## Configuration

Optional environment variables:

```bash
DUNE_BIN_DIR=/home/dune/.dune/bin
DUNE_SERVICE_DB_PATH=/home/dune/.dune/state/server-management-service.sqlite
DUNE_SERVICE_TIME_ZONE=Europe/Amsterdam
DUNE_RESTART_TIME_ZONE=Europe/Amsterdam
DUNE_DASHBOARD_HOST=127.0.0.1
DUNE_DASHBOARD_PORT=8787
DUNE_COMMAND_AUTH_TOKEN_FILE=/home/dune/.dune/state/command-auth-token
```

`DUNE_COMMAND_AUTH_TOKEN` can also be set directly, but a private token file is preferred.

`DUNE_SERVICE_TIME_ZONE` controls the scheduler's wall-clock time. The packaged default is `Europe/Amsterdam` because that is the operator's current server timezone choice. Other operators should set it to their own IANA timezone, for example `Europe/Berlin`, `America/New_York`, or `Australia/Sydney`.

`DUNE_RESTART_TIME_ZONE` is optional. If unset, restart warning payloads use `DUNE_SERVICE_TIME_ZONE`. Set it only if the restart broadcast timestamp should be computed in a different timezone than the service scheduler.

Do not commit `.env`, command-auth tokens, private keys, host-specific IPs, or BattleGroup identifiers.

## Local Development

From this repo on the operator machine:

```powershell
npm install
npm run check
npm run build
node dist/index.js --list
node dist/index.js --once backup
```

Live mode intentionally refuses to run on Windows.

```powershell
node dist/index.js --once backup --run
```

That should fail on Windows with:

```text
Live mode must run on the Linux Dune server, not from Windows.
```

## Full Server Installation

Run these steps on the Dune server as a user with sudo access. The service itself should run as `dune`.

### 1. Install Node.js

Install Node 24 from NodeSource:

```bash
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg.tmp
sudo mv /etc/apt/keyrings/nodesource.gpg.tmp /etc/apt/keyrings/nodesource.gpg
sudo chmod 0644 /etc/apt/keyrings/nodesource.gpg
echo 'deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_24.x nodistro main' \
  | sudo tee /etc/apt/sources.list.d/nodesource.list >/dev/null
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nodejs
```

Verify:

```bash
node --version
npm --version
node --input-type=module -e "import { DatabaseSync } from 'node:sqlite'; const db = new DatabaseSync('/tmp/sms-sqlite-test.sqlite'); db.exec('create table if not exists t (id integer)'); db.close(); console.log('sqlite ok')"
```

### 2. Build The Service

On the machine where the repo is checked out:

```bash
npm install
npm run check
npm run build
```

Copy these paths to the server:

```text
package.json
package-lock.json
README.md
dist/
scripts/
systemd/
```

Example destination:

```text
/tmp/server-management-service-deploy
```

### 3. Install Under /opt

On the server:

```bash
stamp=$(date +%Y%m%d-%H%M%S)
if [ -d /opt/server-management-service ]; then
  sudo mv /opt/server-management-service "/opt/server-management-service.$stamp.bak"
fi

sudo mkdir -p /opt/server-management-service
sudo cp -a /tmp/server-management-service-deploy/. /opt/server-management-service/
sudo chown -R dune:dune /opt/server-management-service
```

### 4. Install Script Dependencies

The service expects its helper scripts in `/home/dune/.dune/bin`.

```bash
cd /opt/server-management-service
sudo -u dune bash scripts/install-script-deps.sh /home/dune/.dune/bin
```

Verify dry-runs:

```bash
/home/dune/.dune/bin/cron-battlegroup-backup --help 2>/dev/null || true
/home/dune/.dune/bin/daily-battlegroup-restart --dry-run
target=$(TZ="${DUNE_RESTART_TIME_ZONE:-${DUNE_SERVICE_TIME_ZONE:-Europe/Amsterdam}}" date -d 'tomorrow 05:00' +%s)
/home/dune/.dune/bin/send-dune-shutdown-broadcast --timestamp "$target" --dry-run
```

### 5. Install The Private Command Token

The generic and shutdown broadcast scripts need the private Dune server command-auth token. Store it on the server, outside the repo:

```bash
install -d -m 0700 /home/dune/.dune/state
sudo -u dune install -m 0600 /dev/null /home/dune/.dune/state/command-auth-token
sudo -u dune editor /home/dune/.dune/state/command-auth-token
```

The file should contain only the token value, with no quotes and no extra text.

Verify that the file exists without printing it:

```bash
sudo -u dune test -s /home/dune/.dune/state/command-auth-token
sudo -u dune stat -c '%a %U %G %n' /home/dune/.dune/state/command-auth-token
```

Expected permissions:

```text
600 dune dune /home/dune/.dune/state/command-auth-token
```

### 6. Dry-Run The Service On The Server

```bash
cd /opt/server-management-service
sudo -u dune node dist/index.js --list
sudo -u dune node dist/index.js --once backup
```

The dry-run backup should print:

```text
[dry-run] /home/dune/.dune/bin/cron-battlegroup-backup
```

### 7. Install systemd

```bash
sudo cp /opt/server-management-service/systemd/server-management-service.service \
  /etc/systemd/system/server-management-service.service
sudo systemctl daemon-reload
sudo systemctl enable server-management-service.service
sudo systemctl start server-management-service.service
```

Verify:

```bash
sudo systemctl --no-pager --full status server-management-service.service
sudo journalctl -u server-management-service.service -n 80 --no-pager
```

Expected log lines include:

```text
Starting Dune server management service (live mode).
Database: /home/dune/.dune/state/server-management-service.sqlite
Dashboard listening on http://127.0.0.1:8787
Scheduled backup every 7200s.
Scheduled update-check every 900s.
Scheduled update-apply every 60s.
Scheduled restart-notice for ...
Scheduled restart for ...
```

If the server should use another timezone, create `/etc/server-management-service.env` before starting the service:

```bash
sudo tee /etc/server-management-service.env >/dev/null <<'EOF'
DUNE_SERVICE_TIME_ZONE=Europe/London
DUNE_RESTART_TIME_ZONE=Europe/London
EOF
sudo chown root:root /etc/server-management-service.env
sudo chmod 0644 /etc/server-management-service.env
```

### 8. Disable Replaced Cron Entries

Only do this after systemd is active.

```bash
mkdir -p /home/dune/.dune/state
stamp=$(date +%Y%m%d-%H%M%S)
crontab -l > "/home/dune/.dune/state/crontab-before-server-management-service-$stamp.txt" 2>/dev/null || true
crontab -l 2>/dev/null \
  | grep -Ev 'cron-battlegroup-backup|cron-battlegroup-update-check|apply-pending-battlegroup-update|daily-battlegroup-restart-notice|daily-battlegroup-restart' \
  | grep -Ev '^# Dune daily restart\.|^# Warning is sent for the next Europe/Amsterdam 05:00 restart target\.|^# Dune update automation:' \
  | crontab - 2>/dev/null || crontab -r 2>/dev/null || true
```

Verify:

```bash
crontab -l 2>/dev/null || echo '<empty>'
```

## Dashboard Access

The dashboard binds to localhost by default.

From the operator machine, use an SSH tunnel:

```powershell
$ssh = 'C:\Windows\System32\OpenSSH\ssh.exe'
& $ssh -N -L 8787:127.0.0.1:8787 dune@<server-host>
```

Then open:

```text
http://127.0.0.1:8787
```

Useful API checks on the server:

```bash
curl -fsS http://127.0.0.1:8787/api/health
curl -fsS 'http://127.0.0.1:8787/api/runs?limit=5'
curl -fsS 'http://127.0.0.1:8787/api/logs?limit=20'
```

## Operational Commands

Run one task manually, dry-run:

```bash
cd /opt/server-management-service
sudo -u dune node dist/index.js --once backup
```

Run one task manually, live:

```bash
cd /opt/server-management-service
sudo -u dune node dist/index.js --once backup --run
```

Restart the daemon:

```bash
sudo systemctl restart server-management-service.service
```

Watch logs:

```bash
sudo journalctl -u server-management-service.service -f
```

Inspect SQLite history:

```bash
ls -lh /home/dune/.dune/state/server-management-service.sqlite
curl -fsS 'http://127.0.0.1:8787/api/runs?limit=10'
```

## Update Or Roll Back Deployment

Before replacing `/opt/server-management-service`, stop the daemon and keep a rollback copy:

```bash
sudo systemctl stop server-management-service.service
stamp=$(date +%Y%m%d-%H%M%S)
sudo mv /opt/server-management-service "/opt/server-management-service.$stamp.bak"
sudo mkdir -p /opt/server-management-service
sudo cp -a /tmp/server-management-service-deploy/. /opt/server-management-service/
sudo chown -R dune:dune /opt/server-management-service
sudo systemctl start server-management-service.service
```

Rollback:

```bash
sudo systemctl stop server-management-service.service
sudo rm -rf /opt/server-management-service
sudo mv /opt/server-management-service.<timestamp>.bak /opt/server-management-service
sudo systemctl start server-management-service.service
```
