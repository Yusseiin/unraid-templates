# Sure on Unraid – step-by-step setup

[Sure](https://github.com/we-promise/sure) is a self-hosted personal finance app. On Unraid it runs as **four containers**, all available from this repository and already configured to talk to each other:

| # | Container | What it does |
|---|-----------|--------------|
| 1 | `postgres17_sure` | Database (PostgreSQL 17 + pgvector) |
| 2 | `redis_sure` | Job queue and cache |
| 3 | `Sure` | Web interface (port 3000), sets up the database on start |
| 4 | `Sure-Worker` | Background jobs: bank syncs, imports, market data, emails |

They all join a private Docker network called `sure-net`, so they find each other by name and the database and Redis are never exposed to your LAN.

> **Do not rename `postgres17_sure` or `redis_sure`.** Sure and Sure-Worker connect to them by those names.

---

## Step 1 – Keep custom networks across reboots

1. Go to **Settings → Docker**.
2. Switch to **Advanced View** (top right).
3. Set **Enable Docker** to **No** and click **Apply** (Docker must be stopped to change the next option).
4. Set **Preserve user defined networks** to **Yes**.
5. Set **Enable Docker** back to **Yes** and click **Apply**.

Without this, Unraid can delete `sure-net` on reboot and the four containers will not start.

## Step 2 – Create the network and the storage folder

Open the Unraid terminal (the `>_` icon top right) and run:

```bash
docker network create sure-net
mkdir -p /mnt/user/appdata/sure/storage
chown -R 1000:1000 /mnt/user/appdata/sure
```

Sure runs as user `1000`, so its storage folder must belong to `1000:1000`.

## Step 3 – Generate your secrets

Still in the terminal, generate two values and **save them somewhere safe** (e.g. your password manager):

```bash
# Secret Key Base for Sure and Sure-Worker
openssl rand -hex 64
```

```bash
# Database password for postgres17_sure, Sure and Sure-Worker
openssl rand -hex 24
```

> **Never lose or change the Secret Key Base.** Sure uses it to encrypt data such as API keys and bank provider credentials. If it changes, that data can no longer be read.

## Step 4 – Install `postgres17_sure`

1. In **Apps**, search for `postgres17_sure` and click **Install**.
2. Paste the **database password** from step 3 into **Database Password**.
3. Leave **Database Name** (`sure_production`) and **Database User** (`sure_user`) as they are.
4. Click **Apply**.
5. Open the container's **Logs** and wait for `database system is ready to accept connections`.

> The user, password and database are only created the **first time** the container starts with an empty data folder. Changing them in the template later does nothing. To start over, stop the container and empty `/mnt/user/appdata/postgres17_sure`.

## Step 5 – Install `redis_sure`

1. Search for `redis_sure` and click **Install**.
2. Nothing to fill in. Click **Apply**.

## Step 6 – Install `Sure`

1. Search for `Sure` (the template by Yusseiin) and click **Install**.
2. Fill in:
   - **Secret Key Base**: the 128-character value from step 3.
   - **Database Password**: the same password you used in `postgres17_sure`.
3. Leave **Database Host** (`postgres17_sure`), **Redis URL** (`redis://redis_sure:6379/1`) and **Storage** as they are.
4. Optional: set **Market Data Sync Timezone** to your timezone, e.g. `Europe/Rome`.
5. Click **Apply**.
6. Open the **Logs**. The first start sets up the database and can take a minute. It is ready when you see Puma listening on port 3000.

## Step 7 – Install `Sure-Worker`

1. Search for `Sure-Worker` and click **Install**.
2. Fill in **exactly the same values** as in Sure:
   - **Secret Key Base**
   - **Database Password**
   - the **Encryption** keys, if you set them in Sure (see [Optional settings](#optional-settings))
   - **Market Data Sync Timezone**, if you changed it
3. Optional: fill in the **SMTP** fields to send emails (only the worker sends email).
4. Click **Apply**.

## Step 8 – Create your account

1. Open `http://YOUR-UNRAID-IP:3000` (or click **WebUI** on the Sure container).
2. Click **Create your account** and register. The first account becomes the **super admin**.
3. Go to **Settings → Self-Hosting → Onboarding** and set it to **Closed** (or **Invite-only**) so nobody else can sign up.

Done! 🎉

---

## Optional settings

| Setting | Where | Notes |
|---------|-------|-------|
| **Assume SSL** = `true` | Sure and Sure-Worker | When Sure is behind an HTTPS reverse proxy (SWAG, Nginx Proxy Manager, Cloudflare Tunnel…). |
| **App Domain** | Sure and Sure-Worker | e.g. `sure.example.com`, used for links in emails. |
| **SMTP** fields + **Email Sender** | Sure-Worker | Password resets and rule notifications. Port `465` with TLS Enabled `true`, or port `587` with TLS Enabled `false`. |
| **Encryption** keys (3 values) | Sure and Sure-Worker | Optional. Blank = derived from the Secret Key Base. If you set them: set **all three**, use the **same** values in both containers, set them **before** the first start and never change them. Generate each with `openssl rand -hex 32`. |
| **Market Data Sync Timezone** | Sure and Sure-Worker | Same value in both. Values saved in Settings → Self-Hosting take precedence. |
| **OpenAI** fields | Sure and Sure-Worker | Optional AI features. Works with OpenAI or any OpenAI-compatible API (e.g. Ollama). |
| **Assistant Type** | Sure and Sure-Worker | Same value in both. Blank = each family picks in Settings → Self-Hosting. `builtin` = in-app chat. `external` = your own agent (needs `EXTERNAL_ASSISTANT_URL` / `EXTERNAL_ASSISTANT_TOKEN`). `mcp_only` = **no in-app AI chat at all** (sidebar, toggle and mobile tab hidden); MCP stays available and background categorization keeps working. See [discussion #203](https://github.com/we-promise/sure/discussions/203). |

## Updating

Update **Sure** first (it migrates the database on start), then **Sure-Worker**. Both must run the same image tag. `redis_sure` can be updated any time. `postgres17_sure` stays on PostgreSQL 17; moving to a newer major version is a manual upgrade.

## Backups

Back up these folders together, ideally with the containers stopped:

- `/mnt/user/appdata/postgres17_sure`
- `/mnt/user/appdata/sure`

Keep your **Secret Key Base** (and encryption keys, if set) with the backup. Without them, a restored database cannot decrypt its data.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Container won't start: `network sure-net not found` | Run `docker network create sure-net` and do step 1. |
| Sure logs show `password authentication failed` | The Database Password in Sure/Sure-Worker doesn't match the one `postgres17_sure` was **first** started with. Use that one, or reset the database (see the note in step 4). |
| Sure logs show `could not translate host name "postgres17_sure"` | The database container was renamed or isn't on `sure-net`. Check its name and Network Type. |
| `Active Record encryption environment variables are partially configured` | Set all three Encryption keys or none. |
| Imports or bank syncs never finish | Sure-Worker isn't running, or its settings don't match Sure. Check its logs. |
| Permission errors on `/rails/storage` | Run `chown -R 1000:1000 /mnt/user/appdata/sure`. |

## Using your own PostgreSQL / Redis

You can skip `postgres17_sure` and `redis_sure`. In Sure and Sure-Worker:

- set **Network Type** to the network your database is on (or `bridge`), and
- change **Database Host** and **Redis URL** to that container's name or your Unraid IP.

The database user needs permission to create the `sure_production` database, or you create it yourself first.

---

Problems with the app itself: [Sure issues on GitHub](https://github.com/we-promise/sure/issues). Problems with these templates: [unraid-templates issues](https://github.com/Yusseiin/unraid-templates/issues).
