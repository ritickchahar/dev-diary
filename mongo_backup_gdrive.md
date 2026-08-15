# MongoDB Backup to Google Drive (Daily, Encrypted)

Nightly off-box backup of a Dockerised MongoDB (plus a SQLite file) to Google Drive,
encrypted before upload. Linux VPS + Windows desktop for the browser step.

> **Why off-box:** a backup on the same disk as the database survives a bad migration,
> not a dead server or a lost hosting account.

---

## 1. Install the tools on the VPS

```bash
sudo apt update && sudo apt install -y rclone gpg
```

`rclone` talks to Drive, `gpg` encrypts the archive before it leaves the box.

---

## 2. Copy the backup script up

From your desktop, in the folder holding the script:

```powershell
ssh <user>@<VPS_IP> "mkdir -p /opt/thunder/scripts"
scp scripts\backup-to-gdrive.sh <user>@<VPS_IP>:/opt/thunder/scripts/
```

Then on the VPS:

```bash
sed -i 's/\r$//' /opt/thunder/scripts/backup-to-gdrive.sh
chmod +x /opt/thunder/scripts/backup-to-gdrive.sh
```

> **Gotcha:** `scp` from Windows carries CRLF line endings across. Without the `sed`,
> bash fails with `\r: command not found`, which reads like the script is corrupt.

---

## 3. SSH back in with a port-forward

```powershell
ssh -L 53682:localhost:53682 <user>@<VPS_IP>
```

Google's OAuth redirect lands on `127.0.0.1:53682`. The VPS has no browser, so this
forwards that port to your desktop and lets you use the normal sign-in flow. Keep the
window open for the whole of step 4.

> Without the tunnel you would install rclone on Windows too and copy an auth token
> between machines. This is fewer moving parts.

---

## 4. Connect Google Drive

```bash
rclone config
```

Answer:

| Prompt | Answer |
|--------|--------|
| `n/s/q` | `n` |
| name | `gdrive` |
| Storage | `drive` |
| client_id | Enter (blank) |
| client_secret | Enter (blank) |
| scope | `3` |
| service_account_file | Enter (blank) |
| Edit advanced config? | `n` |
| Use auto config? | `y` |
| Configure as Shared Drive? | `n` |
| Keep this remote? | `y` |

Then `q` to quit.

- **client_id blank** uses rclone's built-in OAuth client. Fine for one file a night.
- **scope 3 (`drive.file`)** limits the token to files rclone itself creates, so a
  compromised VPS cannot read the rest of your Drive.
- **service_account_file blank**: service accounts have no Drive quota of their own on a
  personal Google account, so uploads fail even though auth succeeds.

4.1. It prints a link. Open it in your **desktop browser** (not PowerShell):

```powershell
Start-Process "http://127.0.0.1:53682/auth?state=<code>"
```

4.2. If Google warns "hasn't verified this app", click **Advanced** then **Go to rclone
(unsafe)**. Expected, it is rclone's shared client.

4.3. Verify:

```bash
rclone mkdir gdrive:thunder-backups
rclone lsd gdrive:
```

---

## 5. Create the encryption passphrase

```bash
sudo openssl rand -base64 32 | sudo tee /root/.halo-backup-pass
sudo chmod 600 /root/.halo-backup-pass
```

Copy the printed value into a password manager **now**.

> **No recovery path:** if this file and the server are lost together, every archive on
> Drive is permanently undecryptable.

---

## 6. Confirm the Docker volume name

```bash
docker volume ls | grep app-data
```

Must match `APP_VOLUME` at the top of the script (default `thunder_app-data`).

---

## 7. Give root the rclone config

```bash
sudo mkdir -p /root/.config/rclone
sudo cp /home/<user>/.config/rclone/rclone.conf /root/.config/rclone/
sudo chmod 600 /root/.config/rclone/rclone.conf
```

You authorised as your own user; cron runs the script as root, which looks in
`/root/.config/`. Skip this and it fails with `didn't find section in config file`.

---

## 8. Run it once by hand

```bash
sudo /opt/thunder/scripts/backup-to-gdrive.sh
```

Success ends with:

```
[...] mongo dump ok (NNNNNNN bytes)
[...] sqlite snapshot ok
[...] upload verified
[...] done: halo-YYYY-MM-DD.tar.gpg (NNNNNNN bytes)
```

---

## 9. Schedule it

```bash
sudo crontab -e
```

Pick nano (option 1) if asked. Add as the last line, save with Ctrl+O, Enter, Ctrl+X:

```
15 3 * * * /opt/thunder/scripts/backup-to-gdrive.sh >> /var/log/halo-backup.log 2>&1
```

Confirm:

```bash
sudo crontab -l
```

> **Cron uses the server's local timezone**, not UTC. Check with `date` if the server
> is not on UTC.

---

## 10. Prove cron actually fires

Do not wait until tomorrow. Cron runs with a minimal `PATH` and no interactive
environment, the classic reason a script that works by hand does nothing at 3am.

```bash
date                    # note the time
sudo crontab -e         # set the schedule two minutes ahead, e.g. "27 17"
# wait, then:
tail -20 /var/log/halo-backup.log
```

A fresh run ending in `done:` means it works. Change the time back to `15 3`.

If the log is empty, cron could not run the script: use absolute paths inside it, or add
`PATH=/usr/local/bin:/usr/bin:/bin` as the first line of the crontab.

---

## 11. Test a restore

**A backup nobody has restored is a hope, not a backup.** Do this on a scratch box.

```bash
rclone copy gdrive:thunder-backups/halo-YYYY-MM-DD.tar.gpg .
gpg --batch --passphrase-file /root/.halo-backup-pass -d halo-YYYY-MM-DD.tar.gpg | tar -xf -
# -> mongo.gz, sqlite.db.gz

gunzip -c mongo.gz | docker compose exec -T mongo mongorestore --archive --drop
gunzip -c sqlite.db.gz > sl-web-client.db     # copy into the volume with the app stopped
```

> `--drop` replaces each collection as it restores. On a live box that is destructive:
> restore into a scratch stack first.

---

## The script

`backup-to-gdrive.sh`, referenced above. Dumps Mongo, snapshots SQLite, encrypts,
uploads, prunes both ends.

```bash
#!/usr/bin/env bash
set -euo pipefail

STACK_DIR="${STACK_DIR:-/opt/thunder}"
DB_NAME="${DB_NAME:-halo}"
LOCAL_DIR="${LOCAL_DIR:-/opt/backups}"
REMOTE="${REMOTE:-gdrive:thunder-backups}"
PASSPHRASE_FILE="${PASSPHRASE_FILE:-/root/.halo-backup-pass}"
APP_VOLUME="${APP_VOLUME:-thunder_app-data}"
KEEP_LOCAL_DAYS="${KEEP_LOCAL_DAYS:-7}"
KEEP_REMOTE_DAYS="${KEEP_REMOTE_DAYS:-30}"
MIN_DUMP_BYTES="${MIN_DUMP_BYTES:-10240}"

STAMP="$(date -u +%Y-%m-%d)"
WORK="$(mktemp -d)"
trap 'rm -rf "$WORK"' EXIT

log() { echo "[$(date -u +%FT%TZ)] $*"; }
die() { log "FAILED: $*"; exit 1; }

[ -f "$PASSPHRASE_FILE" ] || die "no passphrase file at $PASSPHRASE_FILE"
command -v rclone >/dev/null || die "rclone is not installed"
command -v gpg    >/dev/null || die "gpg is not installed"
mkdir -p "$LOCAL_DIR"
cd "$STACK_DIR"

# 1. Mongo. --archive streams to stdout, so nothing is written inside the container.
log "dumping mongo/$DB_NAME"
docker compose exec -T mongo mongodump --archive --db="$DB_NAME" | gzip > "$WORK/mongo.gz" \
    || die "mongodump failed"

SIZE=$(stat -c%s "$WORK/mongo.gz")
[ "$SIZE" -ge "$MIN_DUMP_BYTES" ] || die "dump is only ${SIZE}B, refusing to ship a probably-empty backup"
log "mongo dump ok (${SIZE} bytes)"

# 2. SQLite. ".backup" not cp: a plain copy of a file the app is writing can be torn.
log "snapshotting sqlite"
if docker run --rm -v "$APP_VOLUME":/data -v "$WORK":/out alpine:3 \
        sh -c 'apk add -q sqlite && sqlite3 /data/sl-web-client.db ".backup /out/sqlite.db"' >/dev/null 2>&1; then
    gzip "$WORK/sqlite.db"
    log "sqlite snapshot ok"
else
    log "WARNING: sqlite snapshot failed (volume $APP_VOLUME missing?), continuing with mongo only"
fi

# 3. Encrypt before it leaves the box.
ARCHIVE="$LOCAL_DIR/halo-$STAMP.tar.gpg"
log "encrypting to $ARCHIVE"
tar -C "$WORK" -cf - . \
  | gpg --batch --yes --symmetric --cipher-algo AES256 \
        --passphrase-file "$PASSPHRASE_FILE" -o "$ARCHIVE" \
  || die "encryption failed"
chmod 600 "$ARCHIVE"

# 4. Upload, then prove it landed rather than trusting the exit code.
log "uploading to $REMOTE"
rclone copy "$ARCHIVE" "$REMOTE/" --no-traverse || die "rclone upload failed"
rclone lsf "$REMOTE/" | grep -qx "$(basename "$ARCHIVE")" || die "upload reported success but the file is not on the remote"
log "upload verified"

# 5. Prune both ends.
find "$LOCAL_DIR" -name 'halo-*.tar.gpg' -mtime "+$KEEP_LOCAL_DAYS" -delete
rclone delete "$REMOTE/" --min-age "${KEEP_REMOTE_DAYS}d" || log "WARNING: remote prune failed"

log "done: $(basename "$ARCHIVE") ($(stat -c%s "$ARCHIVE") bytes)"
```

---

## Notes

- **Encrypt before upload.** A dump carries message history, user IPs, ledgers and
  credential rows. Drive is a third party, so it should only ever hold ciphertext.
- **Never back up your app's encryption keys alongside the data.** Vault ciphertext is
  worthless without the key only while the two live in different places. Keys go in a
  password manager.
- **Exclude regenerable caches** (image/texture directories). They dominate the upload
  and rebuild themselves on demand.
- **Size check before upload.** A dump of a dead database is a few hundred bytes and
  otherwise uploads as a perfectly healthy looking backup.
- **No point-in-time recovery.** A nightly snapshot loses the day's writes. Replica-set
  oplog backups are the answer if that window ever gets too expensive.
