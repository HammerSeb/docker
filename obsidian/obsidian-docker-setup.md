# Self-Hosted Obsidian (Docker) — Setup Guide with Working File Upload

This guide covers a clean install of the `linuxserver/obsidian` Docker image, resource limits appropriate for constrained hosts, and the fix needed to make drag-and-drop / Selkies file uploads land inside your vault instead of vanishing.

**_This guide was created with Claude Sonnet 5 (Medium) --- use with care_**

The example vault is called **Labbook**, adjust this to your vault name. It will only work if you have one vault on this instance.

## Background: why the upload fix is needed

The `linuxserver/obsidian` image uses **Selkies** to stream a virtual desktop to your browser. Selkies hardcodes its file-upload destination to `/config/Desktop`. Obsidian, however, only recognizes attachments that live _inside the vault folder_ (e.g. `/config/Labbook`).

By default these are two separate folders, so uploaded files land in `/config/Desktop` and Obsidian can't find them via `![[filename]]` embeds.

**Important:** a symlink from `Desktop` to your vault's attachments folder does **not** work — Selkies performs a realpath security check and will reject uploads that resolve outside the expected `/config/Desktop` directory, logging an error like:

```
ERROR:data_websocket:Path escape attempt detected: '/config/Desktop/file.jpeg' is outside of '/config/Labbook/Attachments'. Discarding.
```

The fix is a **bind mount**, not a symlink — this makes `/config/Desktop` and your vault's attachments folder the _same real directory_ at the filesystem level, so no path resolution is involved and Selkies' check passes.

---

## 1. Prerequisites

- Docker and Docker Compose installed on the host
- An external Docker network for your reverse proxy (this guide assumes one already exists, named `npm`)
- A reverse proxy (e.g. Nginx Proxy Manager) configured to route your chosen subdomain to the container, with **Basic Auth** enabled at the proxy level for password protection before traffic reaches the container

## 2. Plan your folder structure

Decide your vault name up front — this guide uses `Labbook` as an example. Adjust all paths below if you use a different name.

Target structure on the host:

```
docker/obsidian/
├── docker-compose.yaml
├── .env
└── config/
    ├── Desktop/              # will be bind-mounted to vault attachments
    └── Labbook/              # your vault (created on first run via the Obsidian UI)
        └── Attachments/      # create this after the vault exists
```

## 3. Create the project folder and `.env` file

```bash
mkdir -p ~/docker/obsidian/config
cd ~/docker/obsidian
```

Create `.env`:

```env
CUSTOM_USER=yourusername
PASSWORD=yourpassword
PUID=1000
PGID=1000
TZ=Europe/Berlin
```

Check your host user's UID/GID match these values (important for correct file permissions):

```bash
id $USER
```

If the reported `uid`/`gid` differ from `1000`/`1000`, update `PUID`/`PGID` in `.env` to match.

## 4. Create `docker-compose.yaml`

```yaml
services:
  obsidian:
    image: ghcr.io/linuxserver/obsidian:latest
    container_name: obsidian
    security_opt:
      - no-new-privileges:true
      # - seccomp:unconfined   # only add back if you hit rendering/crash issues
    healthcheck:
      test: timeout 10s bash -c ':> /dev/tcp/127.0.0.1/3000' || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 90s
    mem_limit: 768m
    memswap_limit: 768m
    cpus: 1.0
    shm_size: "512mb"
    volumes:
      - ./config:/config:rw
      - ./config/Labbook/Attachments:/config/Desktop:rw
    env_file:
      - ./.env
    networks:
      - npm
    restart: unless-stopped

networks:
  npm:
    external: true
    name: npm
```

Notes on the settings used here:

- `mem_limit` / `cpus` — tune based on your host's available resources; check real usage with `docker stats obsidian` after some normal use and adjust.
- `shm_size` — keep this comfortably below `mem_limit`; 512mb is normally sufficient for a single-user vault.
- `seccomp:unconfined` — left commented out by default (more secure). Some Electron/Chromium rendering issues may require re-enabling it — test without it first.
- The second `volumes:` line is the fix — it mounts the vault's `Attachments` folder directly onto `/config/Desktop` inside the container.

⚠️ On first run, the `./config/Labbook/Attachments` folder doesn't exist yet, since the vault hasn't been created. Docker will auto-create it as an **empty directory owned by root**, which can cause permission issues. It's cleaner to do the first boot in two stages, below.

## 5. First boot — create the vault before enabling the bind mount

**Stage 1: temporarily comment out the second volume line**

```yaml
volumes:
  - ./config:/config:rw
  # - ./config/Labbook/Attachments:/config/Desktop:rw
```

Start the container:

```bash
docker compose up -d
```

Open the subdomain in your browser (through your reverse proxy), log in with the Basic Auth credentials, then the container's own `CUSTOM_USER`/`PASSWORD` login.

In the Obsidian UI, create a new vault named `Labbook` (or your chosen name) when prompted, stored at `/config/Labbook`.

**Stage 2: create the Attachments folder and enable the bind mount**

Stop the container:

```bash
docker compose stop obsidian
```

Create the attachments folder and fix ownership:

```bash
mkdir -p ./config/Labbook/Attachments
chown -R 1000:1000 ./config/Labbook/Attachments
```

Make sure `Desktop` is a **plain directory**, not a symlink (delete and recreate it if unsure):

```bash
rm -rf ./config/Desktop      # safe: only if it's currently a symlink or empty folder
mkdir ./config/Desktop
chown 1000:1000 ./config/Desktop
```

Now uncomment the second volume line in `docker-compose.yaml`:

```yaml
volumes:
  - ./config:/config:rw
  - ./config/Labbook/Attachments:/config/Desktop:rw
```

Recreate the container so the new mount takes effect (a plain `restart` does **not** pick up compose file changes):

```bash
docker compose down
docker compose up -d
```

## 6. Verify the mount is active

```bash
docker inspect obsidian --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{ "\n" }}{{ end }}'
```

You should see two lines, e.g.:

```
/home/youruser/docker/obsidian/config -> /config
/home/youruser/docker/obsidian/config/Labbook/Attachments -> /config/Desktop
```

## 7. Test the upload

Drag an image into a note in the browser UI, then check both paths — they should show the same file, since they're now the same directory:

```bash
ls -la ./config/Desktop/
ls -la ./config/Labbook/Attachments/
```

In your note, embed it:

```markdown
![[filename.png|400]]
```

If it doesn't appear, tail the logs during an upload attempt to check for errors:

```bash
docker logs -f obsidian
```

## 8. Ongoing maintenance notes

- **Pin the image version** once stable, instead of `:latest`, to avoid unexpected breaking updates:
  ```yaml
  image: ghcr.io/linuxserver/obsidian:version-x.x.x
  ```
- **Back up `./config`** regularly — this holds your entire vault, not just app settings.
- **Multiple vaults:** this bind-mount fix only maps one vault's attachments folder to `/config/Desktop`. If you use more than one vault, uploads will only resolve correctly for whichever vault's `Attachments` folder is mounted.
- **Resource limits:** re-check `docker stats` periodically, especially after adding plugins or growing your vault, and adjust `mem_limit`/`cpus` as needed.
