# VPS Deployment

Deploying a Dockerized app to a VPS with a real domain, automatic TLS, and CI/CD.
Stack: **Contabo VPS + Cloudflare DNS + Caddy + Docker Compose + GitHub Actions + GHCR**.

Every command here was run against a live box. The gotchas at the end are things that
actually broke, not theory.

**The shape:** CI builds the image → pushes it to GHCR → the VPS pulls it.
**The VPS never builds.** An `npm ci` / `dotnet publish` on a small box that is also running
the app and a database is an OOM risk mid-deploy, and it wastes shared vCPU.

---

1. **Buying the VPS**

   1.1. **NVMe vs SSD.** If the app's disk workload is small random writes (SQLite, Mongo,
   Postgres) rather than bulk storage, take the smaller **NVMe** over the larger SSD. Latency
   per operation matters; capacity usually doesn't. Disk is also the one spec you can't upgrade
   without rebuilding the VPS.

   1.2. **Contabo has no coupon codes.** Discounts are applied automatically for longer billing
   terms. Check the "Discounted Outlet Servers" section before configuring a new one.

2. **DNS (Cloudflare)**

   2.1. Delete any parking records the registrar left behind (often AWS IPs).

   2.2. Add two A records, **proxy OFF (grey cloud)**:

   | Type | Name | Content | Proxy status | TTL |
   |------|------|---------|--------------|-----|
   | A    | `@`  | VPS IPv4 | **DNS only** | Auto |
   | A    | `www`| VPS IPv4 | **DNS only** | Auto |

   2.3. **The grey cloud is load-bearing.** With the orange cloud on, Cloudflare answers DNS with
   its own edge IPs and terminates TLS itself, so Caddy's Let's Encrypt HTTP-01 challenge fails.
   In "Flexible" SSL mode you then get an infinite redirect loop that looks exactly like an app bug.

   2.4. Set **SSL/TLS → Overview → Full (strict)** anyway. It's inert while DNS-only, but it can
   never silently fall back to Flexible if you enable the proxy later.

   2.5. Delete AAAA records unless IPv6 genuinely points at the box — Let's Encrypt may validate
   over IPv6 and fail even when IPv4 is perfect.

   2.6. Verify. Should return **only** the VPS IP (no `104.21.x`, `172.67.x`, `2606:4700:` —
   those are Cloudflare's):

   ```powershell
   nslookup yourdomain.com 1.1.1.1
   nslookup www.yourdomain.com 1.1.1.1
   ```

3. **Host prep** (SSH in as root)

   3.1. A non-root user:

   ```bash
   adduser deploy
   usermod -aG sudo deploy
   apt update && apt upgrade -y
   ```

   3.2. Swap. It doesn't make anything faster — it turns an OOM *kill* into degraded performance
   you can see coming:

   ```bash
   fallocate -l 4G /swapfile
   chmod 600 /swapfile
   mkswap /swapfile
   swapon /swapfile
   echo '/swapfile none swap sw 0 0' >> /etc/fstab
   free -h
   ```

   3.3. Firewall. Use `--force` so the confirmation prompt can't swallow your next pasted line:

   ```bash
   ufw allow OpenSSH
   ufw allow 80
   ufw allow 443
   ufw --force enable
   ufw status
   ```

   3.4. Docker:

   ```bash
   curl -fsSL https://get.docker.com | sh
   usermod -aG docker deploy
   su - deploy
   docker ps      # must work WITHOUT sudo — proves the group took
   ```

4. **The deploy directory**

   Keep exactly three files on the box, none of them in git: `.env`, `Caddyfile`,
   `docker-compose.yml`.

   4.1. Create it:

   ```bash
   sudo mkdir -p /opt/app && sudo chown deploy:deploy /opt/app
   cd /opt/app
   ```

   4.2. `.env` — generate secrets inline (`<<EOF` unquoted so bash expands `$(...)`):

   ```bash
   cat > .env <<EOF
   ADMIN_USERNAME=admin
   ADMIN_PASSWORD=$(openssl rand -base64 24)
   SECRET_KEY=$(openssl rand -base64 32)
   EOF
   chmod 600 .env
   cat .env        # copy these somewhere safe NOW
   ```

   4.3. `Caddyfile` — `<<'EOF'` **quoted**, so bash doesn't mangle Caddy's `{uri}` placeholder:

   ```bash
   cat > Caddyfile <<'EOF'
   yourdomain.com {
   	reverse_proxy app:8080
   }

   www.yourdomain.com {
   	redir https://yourdomain.com{uri} permanent
   }
   EOF
   ```

   4.4. `docker-compose.yml` — **do not paste this one.** Long heredocs get mangled over SSH.
   Copy it up from the dev machine instead:

   ```powershell
   scp docker-compose.prod.yml deploy@<VPS_IP>:/opt/app/docker-compose.yml
   ```

   4.5. Verify both files at once. `config` parses the YAML **and** substitutes `.env`:

   ```bash
   docker compose config
   ```

   Use `${VAR:?message}` in compose for required secrets — the stack then refuses to start with a
   clear error instead of booting with a weak default.

5. **Pull the image and start**

   5.1. GHCR packages inherit the repo's visibility, so a private repo needs a token to pull.
   On GitHub: Settings → Developer settings → Personal access tokens → **Tokens (classic)** →
   tick **only `read:packages`**.

   ```bash
   docker login ghcr.io -u <github-user>     # paste the token at the Password: prompt
   docker pull ghcr.io/<github-user>/<repo>:latest
   docker compose up -d
   docker compose ps
   ```

   5.2. Caddy requests the certificate on the first request:

   ```bash
   docker compose logs --tail=30 caddy      # want: "certificate obtained successfully"
   curl -s https://yourdomain.com/ready
   ```

   One healthy `/ready` proves TLS + reverse proxy + app + database in a single response.

6. **TLS — nothing to buy, nothing to renew**

   6.1. Caddy gets real Let's Encrypt certs and **renews automatically** ~30 days before expiry.
   No cron, no downtime.

   6.2. Certs live in the `caddy-data` volume, so redeploys don't touch them.

   6.3. **Never `docker compose down -v`.** The `-v` deletes volumes: certs, database, and any
   secrets-at-rest, all at once.

   6.4. Let's Encrypt allows **5 certs per domain per week**. Repeatedly destroying `caddy-data`
   while debugging can lock you out for a week.

7. **CI/CD with GitHub Actions**

   7.1. Three workflows, all scoped to `main`:

   | File | Fires on | Does |
   |------|----------|------|
   | `deploy.yml` | push to `main` | build image → GHCR → SSH to VPS → `compose pull` + `up -d` → poll `/ready` |
   | `ci.yml` | PR **into** `main` | typecheck + build |
   | `test.yml` | push to `main` | smoke test that Actions runs at all |

   7.2. Don't run the typecheck workflow on pushes to `main` as well — the deploy workflow already
   compiles the same code inside the Docker build. That just doubles the minute burn.

   7.3. Give every workflow `workflow_dispatch` so it can be re-run from the Actions tab without
   inventing a dummy commit.

   7.4. Repo secrets the deploy job needs:

   | Secret | Value |
   |--------|-------|
   | `VPS_HOST` | VPS IPv4 |
   | `VPS_USER` | `deploy` |
   | `VPS_SSH_KEY` | private key whose public half is in the VPS's `authorized_keys` |
   | `VPS_PATH` | `/opt/app` |
   | `APP_URL` | `yourdomain.com` |

   7.5. Make the deploy job **health-gate**: poll `/ready` for ~2 min and, on failure, dump the
   container logs and exit non-zero. A broken deploy should be loud, not a silently dead site.

   7.6. Tag every build with **both** `:latest` and `:<git-sha>`. The sha tag is the rollback handle:
   point the compose image at `:<good-sha>` and `docker compose up -d`.

8. **Free-tier Actions facts**

   | Repo visibility | Minutes | Card required |
   |-----------------|---------|---------------|
   | Public  | Unlimited | No |
   | Private | 2,000/month | No |

   A new account with no payment method is fine. Only a **billing lock** on an existing account
   blocks Actions — and it's account-wide, so it blocks every repo, and going public does **not**
   clear it:

   ```
   GitHub Actions workflows can't be executed on this repository.
   Your account's billing is currently locked.
   ```

   Fix at https://github.com/settings/billing. While there, check the **Actions spending limit** —
   if it's $0 and the private-repo minutes are spent, runs stay blocked even after billing unlocks.

9. **Mirroring a repo to a second account** (the billing-lock workaround)

   Run CI on a second account by mirroring the repo there.

   9.1. Create an **empty** repo on the second account (no README — a commit there diverges the
   histories).

   9.2. Dual-push remote: one `git push` writes to both. **Adding a push URL replaces the implicit
   default**, so the original must be re-added explicitly first — this is the step people get wrong:

   ```bash
   git remote set-url --add --push origin https://github.com/<primary>/<repo>.git
   git remote set-url --add --push origin https://github.com/<mirror>/<repo>.git
   git remote -v     # expect 1 (fetch) line and 2 (push) lines
   ```

   9.3. If it errors with `remote.origin.pushurl has multiple values`, clear them and redo both:

   ```bash
   git config --unset-all remote.origin.pushurl
   ```

   9.4. One-time seed of the full history:

   ```bash
   git push --all  https://github.com/<mirror>/<repo>.git
   git push --tags https://github.com/<mirror>/<repo>.git
   ```

   9.5. **Caveat:** dual-push only mirrors what *you* push from this machine. A PR merged through
   GitHub's web UI on one account won't reach the other until you pull and push again.

   9.6. **GHCR ownership follows the account that pushes the image.** If CI runs on the mirror, the
   image is `ghcr.io/<mirror>/<repo>` — that's what compose must reference and what the VPS logs
   into.

   9.7. **Secrets do not mirror.** Add them again on the mirror repo.

10. **Ops**

    10.1. Every deploy restarts the container and drops whatever was in memory. Deploy at quiet hours.

    10.2. Nightly database backup:

    ```bash
    0 3 * * * cd /opt/app && docker compose exec -T mongo mongodump --archive --db=<db> | gzip > /opt/backups/<db>-$(date +\%F).gz
    ```

    10.3. Prune any unbounded on-disk cache the app writes:

    ```bash
    0 4 * * 0 cd /opt/app && docker compose exec -T app find /data/cache -type f -mtime +30 -delete
    ```

    10.4. Expose an endpoint that reports process RSS, and size the app's concurrency limit from a
    real soak rather than a guessed default:

    ```
    limit = floor((AvailableMb × 0.70 − IdleRssMb) / MarginalMbPerUnit)
    ```

11. **Gotchas that actually bit**

    11.1. **`ufw enable`'s prompt swallows your next pasted line.** Paste a block and the "may
    disrupt SSH" prompt eats the following command as its answer, and the firewall silently aborts.
    Use `ufw --force enable`, or paste one line at a time.

    11.2. **Long heredocs get mangled over SSH.** A pasted `docker-compose.yml` arrived corrupted and
    truncated. `scp` files instead of pasting them.

    11.3. **`www` typo'd as `ww`.** Cloudflare truncates the Name column from the *right*, so
    `ww.example.com` and `www.example.com` look nearly identical in the record list.

    11.4. **`docker login ghcr.io` failing with `denied`** is usually a partial token paste — the
    token doesn't echo, so you can't see the truncation. Just retry.

    11.5. **Databases in compose need no auth *if* they're `expose`d only.** Never add a `ports:`
    mapping to the database service — that puts an unauthenticated DB on the public internet.

    11.6. **Mirror the right repo.** Pushing a private codebase to the wrong account means deleting
    the whole repo, not just the branches — objects stay reachable by SHA.
