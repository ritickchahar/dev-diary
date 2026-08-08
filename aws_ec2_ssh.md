# AWS EC2 — SSH with a PEM key

Connecting to an EC2 box when all you have is a `.pem` key and an IP. Windows
paths shown for Git Bash + PowerShell.

1. **Keep keys in one place**

   Store PEM keys in your user SSH folder so `ssh` and the `config` file can find them:

   ```bash
   C:\Users\<you>\.ssh\
   ```

   > Note: when saving a pasted key in Notepad, set **Save as type: All Files** so
   > it isn't silently saved as `key.pem.txt`. The file must start with
   > `-----BEGIN ... PRIVATE KEY-----` and end with `-----END ... PRIVATE KEY-----`.

2. **Fix key permissions (required)**

   SSH refuses a key readable by others.

   2.1. PowerShell (Windows ACLs — what `ssh.exe` checks):

   ```powershell
   icacls "C:\Users\<you>\.ssh\<key>.pem" /inheritance:r
   icacls "C:\Users\<you>\.ssh\<key>.pem" /grant:r "$($env:USERNAME):(R)"
   ```

   2.2. Git Bash / Linux / macOS:

   ```bash
   chmod 400 ~/.ssh/<key>.pem
   ```

3. **Find the right username**

   The IP + key are not enough — the login user depends on the AMI. A wrong user
   gives `Permission denied (publickey)` even with the correct key.

   | AMI            | Default user |
   |----------------|--------------|
   | Ubuntu         | `ubuntu`     |
   | Amazon Linux   | `ec2-user`   |
   | Debian         | `admin`      |
   | RHEL           | `ec2-user` / `root` |
   | Bitnami        | `bitnami`    |

   3.1. Probe non-interactively (no hang, changes nothing on the server):

   ```bash
   for u in ubuntu ec2-user admin; do
     ssh -i ~/.ssh/<key>.pem -o BatchMode=yes -o StrictHostKeyChecking=accept-new \
       -o ConnectTimeout=10 "$u@<IP>" whoami && { echo "user=$u"; break; }
   done
   ```

4. **Connect**

   ```bash
   ssh -i "C:\Users\<you>\.ssh\<key>.pem" <user>@<IP>
   ```

5. **Save a shortcut in `~/.ssh/config`**

   Append a block so you can connect by name:

   ```
   Host <alias>
       HostName <IP>
       User <user>
       IdentityFile ~/.ssh/<key>.pem
   ```

   Then just:

   ```bash
   ssh <alias>
   ```

6. **Troubleshooting**

   - `Permission denied (publickey)` → wrong username, or wrong key for this instance.
   - Connection hangs / times out → the instance **security group** must allow inbound
     TCP **22** from your IP.
   - `UNPROTECTED PRIVATE KEY FILE` → redo step 2.
   - First connect asks to confirm host authenticity → type `yes` (normal).

---

## Worked example — finminos backend

- **Key:** `C:\Users\<you>\.ssh\finminos-backend-key.pem`
- **IP:** `13.127.174.105`
- **User:** `ubuntu` (Ubuntu AMI — `ec2-user` was refused)

`~/.ssh/config` entry:

```
Host finminos
    HostName 13.127.174.105
    User ubuntu
    IdentityFile ~/.ssh/finminos-backend-key.pem
```

Connect:

```bash
ssh finminos
```

### Once on the box: free disk space safely

Disk was at 98% (`df -h /`). The safe, high-impact win is clearing Docker
**build cache** — it only speeds up future builds, so removing it never touches
the running container or its live image:

```bash
docker system df          # see what's reclaimable
docker builder prune -af  # frees build cache (was ~1.4 GB here)
df -h /                   # confirm
```

> Note: do **not** `docker rmi` the app's live image, and avoid
> `docker volume prune` without checking what's inside (volumes may hold data).
