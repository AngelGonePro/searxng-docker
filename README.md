curl the zip - https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip

curl -L -o searxng-tor.zip https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip
unzip searxng-tor.zip

---

1. nano env — missing the dot. You edited a file named env, but Docker Compose only auto-loads a file literally named .env (hidden file). Your real .env still has the placeholder values, which is why everything's defaulting to blank. Fix:

bash
mv env .env    # if you want to keep what you typed, otherwise just:
nano .env

Then fill in SEARXNG_SECRET (openssl rand -hex 32), SEARXNG_BASE_URL, and PROJECT_DIR=/home/angel/searxng-tor.

2. The updater/ folder doesn't exist on disk. The build error means only docker-compose.yml and .env made it onto this machine — updater/Dockerfile and updater/update.sh (and searxng/settings.yml) never got copied over. Check what's actually there:

bash
ls -la ~/searxng-tor

If updater/ and searxng/ are missing, you'll need to recreate them. Fastest way — from ~/searxng-tor:

List files with line counts for reference
List files with line counts for reference
bash
mkdir -p ~/searxng-tor/searxng ~/searxng-tor/updater

then re-download/re-copy searxng/settings.yml, updater/Dockerfile, and updater/update.sh from the files I generated above into those folders (however you're transferring files onto this box — scp, git, or just recreating them with nano).

Once ls ~/searxng-tor shows .env, docker-compose.yml, searxng/settings.yml, and updater/{Dockerfile,update.sh} all present, and .env has real values, docker compose up -d --build should go clean.

---

mv Dockerfile update.sh updater/

Also — ls doesn't show a searxng/ folder with settings.yml in it either, so that still needs to exist before searxng container will start correctly. And you ran nano env again (no dot) — that stray env file isn't doing anything; safe to rm env once .env has your real values so it doesn't confuse you later.

After the move, confirm the layout matches:

Show expected final layout
Show expected final layout
bash
find . -maxdepth 2 -type f

should print exactly:

./.env
./docker-compose.yml
./searxng/settings.yml
./updater/Dockerfile
./updater/update.sh

If searxng/settings.yml is still missing, recreate it there too (same as before — mkdir already made the folder, just the file needs to land in it). Once all five are in place, docker compose up -d --build again.

---

Full layout:

```
searxng-tor/
├── .env
├── docker-compose.yml
├── update.sh
├── proxy/
│   └── Caddyfile
├── rotator/
│   ├── Dockerfile
│   └── rotate.sh
└── searxng/
    ├── Dockerfile
    ├── entrypoint.sh
    ├── render.py
    ├── inject_style.py
    ├── custom-cosmoscraft.css
    ├── settings.template.yml
    └── limiter.toml
```

Before you run it: set PROJECT_DIR in .env to the actual absolute path where you clone/place this folder on the Linode/host (e.g. /home/angel/searxng-tor). That's required — the updater runs docker compose against the host daemon from inside its own container, so bind mounts have to resolve against real host paths, not container-relative ones. I switched the searxng service's config mount to use ${PROJECT_DIR} too so both invocation paths (you running compose by hand, and the updater running it) resolve identically.

Tuning knobs, also in .env:

UPDATER_CHECK_INTERVAL (default 300s) — how often it polls
UPDATER_IDLE_MINUTES (default 15) — consecutive idle checks required before it updates

docker compose up -d --build on first run to build the updater image alongside the rest.
