curl the zip - https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip

curl -L -o searxng-tor.zip https://raw.githubusercontent.com/AngelGonePro/searxng-docker/refs/heads/main/searxng-tor.zip
unzip searxng-tor.zip

bash
mv env .env    # if you want to keep what you typed, otherwise just:
nano .env

docker compose up -d --build on first run to build the updater image alongside the rest.

find . -maxdepth 2 -type f

should print exactly:

./rotator/Dockerfile
./rotator/rotate.sh
./proxy/Caddyfile
./searxng/Dockerfile
./searxng/settings.template.yml
./searxng/entrypoint.sh
./searxng/inject_style.py
./searxng/render.py
./searxng/limiter.toml
./searxng/custom-cosmoscraft.css
./update.sh
./.env
./docker-compose.yml

---

openssl rand -hex 32

gives you a new SEARXNG_SECRET.

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
