curl the zip - https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip

Full layout:

```
searxng-tor/
├── docker-compose.yml
├── .env
├── searxng/settings.yml
└── updater/
    ├── Dockerfile
    └── update.sh
```

Before you run it: set PROJECT_DIR in .env to the actual absolute path where you clone/place this folder on the Linode/host (e.g. /home/angel/searxng-tor). That's required — the updater runs docker compose against the host daemon from inside its own container, so bind mounts have to resolve against real host paths, not container-relative ones. I switched the searxng service's config mount to use ${PROJECT_DIR} too so both invocation paths (you running compose by hand, and the updater running it) resolve identically.

Tuning knobs, also in .env:

UPDATER_CHECK_INTERVAL (default 300s) — how often it polls
UPDATER_IDLE_MINUTES (default 15) — consecutive idle checks required before it updates

docker compose up -d --build on first run to build the updater image alongside the rest.
