curl the zip - https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip

curl -L -o searxng-tor.zip https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip
unzip searxng-tor.zip

bash
mv env .env    # if you want to keep what you typed, otherwise just:
nano .env

docker compose up -d --build on first run to build the updater image alongside the rest.

find . -maxdepth 2 -type f

should print exactly:

./.env
./docker-compose.yml
./searxng/settings.yml
./updater/Dockerfile
./updater/update.sh

---

openssl rand -hex 32

gives you a new SEARXNG_SECRET.

bash
openssl rand -hex 20

gives you a new TOR_CONTROL_PASSWORD.

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
