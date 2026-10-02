# Running DoneWhen on a Mac Pro with Tailscale

This is a copy-paste recipe for hosting DoneWhen on a Mac Pro behind HTTPS with Tailscale.

It follows the setup described in `docs/SELF-HOSTING.md` and uses the app's default port `:8090` plus a Tailscale HTTPS endpoint.

## 1. Install the tools

```bash
# Install Tailscale
brew install tailscale
```

Make sure Docker Desktop is installed and started. If you use Podman instead, see `docs/PODMAN.md`.

## 2. Clone the repo and create `.env`

```bash
cd ~/code
git clone https://github.com/johnreginald/donewhen.git
cd donewhen
cp .env.example .env
```

Then set the required environment values:

```bash
DONEWHEN_SESSION_SECRET=$(openssl rand -hex 32)
cat > .env <<EOF
DONEWHEN_ENV=prod
DONEWHEN_BASE_URL=https://myserver.tailnet-name.ts.net
DONEWHEN_LISTEN_ADDR=:8080
DONEWHEN_SESSION_SECRET=${DONEWHEN_SESSION_SECRET}
DONEWHEN_ISSUE_PREFIX=R
DONEWHEN_TRUSTED_PROXY_HEADER=
DONEWHEN_VAPID_PUBLIC=
DONEWHEN_VAPID_PRIVATE=
DONEWHEN_VAPID_SUBJECT=mailto:you@example.com
POSTGRES_USER=donewhen
POSTGRES_PASSWORD=change-this-password
POSTGRES_DB=donewhen
DONEWHEN_SITE_ADDRESS=:80
EOF
```

Replace:

- `myserver.tailnet-name.ts.net` with the actual Tailscale HTTPS address created later
- `change-this-password` with a strong Postgres password

## 3. Start Tailscale and expose the app

```bash
sudo tailscale up
# or: tailscale up --authkey=tskey-...  if you use an auth key
```

Then expose the local app port:

```bash
tailscale serve --bg 8090
```

Check the status:

```bash
tailscale serve status
```

You should see a URL similar to:

```text
https://myserver.tailnet-name.ts.net
```

Use that exact URL as `DONEWHEN_BASE_URL` in `.env`.

## 4. Start DoneWhen

```bash
docker compose up -d --build
```

## 5. Create the first user

```bash
docker compose exec donewhen /app/donewhen user you@example.com 'a-strong-password'
```

## 6. Create your first workspace

```bash
docker compose exec donewhen /app/donewhen workspace create "My Work" MYW
```

## 7. Open the app

Open:

```text
https://myserver.tailnet-name.ts.net
```

Sign in with:

- email: `you@example.com`
- password: `a-strong-password`

## 8. Optional: use a better admin password

```bash
docker compose exec donewhen /app/donewhen user you@example.com 'YourRealPassword123!'
```

Then sign in again with the new password.

## 9. Optional: enable Web Push

Generate a VAPID key pair:

```bash
docker compose run --rm donewhen /app/donewhen genvapid
```

Paste the output into `.env`:

```env
DONEWHEN_VAPID_PUBLIC=...
DONEWHEN_VAPID_PRIVATE=...
DONEWHEN_VAPID_SUBJECT=mailto:you@example.com
```

Then restart the app:

```bash
docker compose up -d --build
```

## 10. Useful commands

```bash
# view app logs

docker compose logs -f --tail=100 donewhen

# stop all containers

docker compose down

# restart after config changes

docker compose up -d --build
```

## Notes

- DoneWhen runs on `127.0.0.1:8090` by default, so the app is local to the machine until Tailscale exposes it.
- `DONEWHEN_ENV=prod` is required for secure session cookies and HTTPS behavior.
- The project docs recommend a reverse proxy or private-network HTTPS setup for production use. Tailscale is the simplest choice for a Mac Pro at home or on a private network.
- If you need public internet access instead of Tailscale, use the options in `docs/SELF-HOSTING.md`: Caddy with a domain or a Cloudflare Tunnel.
