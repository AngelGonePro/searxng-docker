# searxng-tor

A self-hosted [SearXNG](https://github.com/searxng/searxng) metasearch instance that sends all engine traffic through Tor, packaged as a Docker Compose stack and configured from a single `.env` file.

## What you get

- **SearXNG** with every working search engine and category (general, images, videos, news, music, files, maps, science, etc.) enabled by default. The engine list is discovered from the SearXNG image itself at every start, so it never goes out of date across versions.
- **Tor** as the outbound proxy (SOCKS5h, so DNS also resolves over Tor). SearXNG fails closed: it will not serve requests if Tor isn't working.
- **Per-engine Tor bypass.** Engines that block Tor outright can connect directly instead, set with one line each in `.env`.
- **Disable engines** by name from `.env`.
- **Caddy reverse proxy** in front of SearXNG so the bot-detection limiter works. It restores the real visitor IP behind Cloudflare Tunnel and/or an upstream nginx.
- **Valkey** (Redis-compatible) for the limiter.
- **Automatic Tor circuit rotation** every few minutes, with no `docker.sock` access needed.
- **Obfs4 bridge support** for networks that block Tor relays.
- **Dark custom theme** and a footer with a link to this repo.
- **Manual, tested updates.** Nothing updates itself.

## Layout

```
searxng-tor/
├── .env                    all settings live here
├── docker-compose.yml
├── update.sh               update SearXNG (tests the new version first)
├── deploy-update.sh        update this stack from GitHub, keeping your .env
├── set-bridges.sh          set Tor bridges without editing files
├── proxy/Caddyfile
├── rotator/                Tor circuit rotator
├── searxng/                SearXNG image customisation, settings template, theme
└── tor/                    Tor image (with obfs4proxy)
```

## Requirements

- Docker with the Compose plugin
- `curl` and `unzip` on the host

## Install

```
curl -L -o searxng-tor.zip https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip && unzip searxng-tor.zip
cd searxng-tor
```

Generate two secrets and put them in `.env`:

```
openssl rand -hex 32     # SEARXNG_SECRET
openssl rand -hex 20     # TOR_CONTROL_PASSWORD
```

Set at least these in `.env`:

| Setting | Purpose |
| --- | --- |
| `SEARXNG_SECRET` | SearXNG secret key |
| `TOR_CONTROL_PASSWORD` | Password the rotator uses on Tor's control port |
| `SEARXNG_BASE_URL` | Public URL of your instance, e.g. `https://search.example.com/` |
| `SEARXNG_PORT` | Port published on the host (Caddy listens here) |

Then start it:

```
docker compose up -d --build
```

The first start builds three small images and can take a few minutes. Open `http://<host>:<SEARXNG_PORT>`.

## Configuration (`.env`)

Everything is controlled from `.env`. Apply changes with `docker compose up -d --build`. Don't put inline comments on value lines (Docker Compose doesn't support them in env files).

| Setting | Description |
| --- | --- |
| `SEARXNG_VERSION` | Pinned SearXNG image version. Changed only by `./update.sh`. |
| `SEARXNG_PORT` | Host port for the instance. |
| `SEARXNG_BASE_URL` | Public URL. |
| `SEARXNG_SECRET` | Secret key. |
| `SEARXNG_LIMITER` | `true` enables bot detection / rate limiting. Leave on for public instances. |
| `SEARXNG_UWSGI_WORKERS`, `SEARXNG_UWSGI_THREADS` | Worker / thread counts. |
| `DISABLED_ENGINES` | Comma-separated engine names to turn off. **One line only.** |
| `TOR_BYPASS_<ENGINE>` | `true` makes that engine connect directly instead of through Tor. |
| `TOR_CONTROL_PASSWORD` | Tor control port password. |
| `TOR_ROTATE_MINUTES` | Minutes between new Tor circuits. |
| `TOR_BRIDGE_1..3` | Optional obfs4 bridges (see below). |
| `UPDATE_HEALTH_WAIT` | Seconds `update.sh` waits before health-checking a test container. |

### Disabling engines

Use the engine names shown in SearXNG's Preferences or engine stats, comma separated, on a single line:

```
DISABLED_ENGINES=brave, startpage, vimeo, yandex
```

Unknown names are reported in the SearXNG container log at startup.

### Bypassing Tor for specific engines

Some engines block Tor exit nodes. For those, set the matching variable to `true`. The name is `TOR_BYPASS_` followed by the engine name in uppercase, with anything that isn't a letter or digit turned into `_`:

```
TOR_BYPASS_GOOGLE_IMAGES=true
TOR_BYPASS_BRAVE_NEWS=true
```

Engines without a variable (or set to `false`) go through Tor. Engines that bypass Tor reveal your server's real IP to that engine, so use this sparingly.

## Updating

### Update this stack (files from GitHub)

```
cd searxng-tor
./deploy-update.sh
```

Downloads the latest release zip, replaces every file **except `.env`**, adds any new settings to your `.env` (existing values are never changed, and a backup is saved as `.env.bak.<timestamp>`), then rebuilds and restarts. To use a different location for the zip:

```
SEARXNG_TOR_ZIP_URL=https://example.com/searxng-tor.zip ./deploy-update.sh
```

Note: it overwrites `docker-compose.yml`, so keep local edits elsewhere.

### Update SearXNG itself

```
./update.sh
```

Pulls the newest SearXNG image, builds a test image, runs it as a disposable container and requires it to stay up and answer HTTP 200. Only then does it update `SEARXNG_VERSION` in `.env` and restart the live stack. If the test fails, the running instance is left untouched.

## Tor bridges

If your network blocks Tor relays, the `tor` container stays stuck at `Bootstrapped 30%` and the site shows **502 Bad Gateway** (SearXNG refuses to serve without a working Tor). Check with:

```
docker compose logs --tail=20 tor
```

Fix it with obfs4 bridges:

1. Get bridge lines (each starts with `obfs4`) from <https://bridges.torproject.org> (choose Obfs4), or email `bridges@torproject.org` from a Gmail address with the body `get transport obfs4`. If your DNS filter blocks Tor Project domains, use a device on a different network or allow `torproject.org`.
2. Run:

```
./set-bridges.sh
```

3. Paste the lines, one per line, then press Enter on an empty line.

The script saves them to `.env`, restarts Tor, waits for `Bootstrapped 100%`, then restarts SearXNG. You can also pass lines as arguments, or remove all bridges with `./set-bridges.sh --clear`. Bridges survive `./deploy-update.sh`.

## Running behind Cloudflare Tunnel and/or nginx

The bundled Caddy proxy trusts private network ranges and, when a request carries `Cf-Connecting-Ip`, uses it as `X-Forwarded-For` so SearXNG's limiter sees the real visitor. This works through an upstream nginx as long as nginx doesn't strip that header.

Point your tunnel or nginx upstream at `http://<host>:<SEARXNG_PORT>`.

**Security:** the proxy trusts `Cf-Connecting-Ip` unconditionally. Only expose the port through your tunnel/proxy, never directly to the internet, or visitors could forge IPs and defeat rate limiting.

## Troubleshooting

| Symptom | Likely cause / fix |
| --- | --- |
| 502 Bad Gateway | SearXNG is still starting (wait ~30 s), or Tor isn't connected. Check `docker compose ps` and `docker compose logs --tail=20 tor`. |
| Tor stuck at 30% | Network blocks Tor relays. Use bridges (above). |
| Many engines fail with timeouts | Tor is slow or down; check Tor first. Add `TOR_BYPASS_*` for engines that block Tor. |
| An engine keeps erroring | Add it to `DISABLED_ENGINES`. |
| `.env` values ignored | Make sure there are no inline comments on value lines and `DISABLED_ENGINES` appears only once. |
| Changes not applied | Run `docker compose up -d --build`. |

Useful commands:

```
docker compose ps
docker compose logs --tail=50 searxng
docker compose logs --tail=20 tor
docker compose up -d --build
```

## Security notes

- Keep `.env` out of public repositories. It holds your secrets. The `.env` in the release zip contains placeholders only; replace them with your own generated values.
- Change `SEARXNG_SECRET` and `TOR_CONTROL_PASSWORD` if they were ever exposed, then run `docker compose up -d --build`.
- Tor protects the origin of engine requests, not your visitors' browsers from your server. Don't treat this as an anonymity service for the people using it.

## Credits

Built on [SearXNG](https://github.com/searxng/searxng), [Tor](https://www.torproject.org/), [Caddy](https://caddyserver.com/) and [Valkey](https://valkey.io/).
