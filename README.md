# homelab-cache

A self-hosted, transparent pull-through cache for container registries, in **one container on port 443**. Clients keep using the real image names; the cache is chosen by the registry's hostname.

```
docker pull ghcr.io/owner/app   ──▶  ghcr.io                ─┐
docker pull quay.io/org/app     ──▶  quay.io                 ├──▶  this container :443  ──▶  cache  ──▶  real registry
docker pull alpine              ──▶  registry-1.docker.io   ─┘      (chosen by TLS server name)
```

The same container can also cache **apt mirrors** (Debian, Ubuntu and friends) on port 80, the same transparent way: clients keep their `sources.list` and DNS sends the mirror names here. See [Caching apt packages](#caching-apt-packages).

A web UI (plain HTTP on `:8080`, and over TLS on any other name or IP) shows cached registries, images and sizes, with light and dark modes.

## What you need

1. **DNS on the clients** that sends each registry name (`ghcr.io`, `quay.io`, …) to this server. See [Client setup](#client-setup-system-wide).
2. **A certificate valid for those names**, trusted by the clients. You can bring your own (below) or let the container generate one.
3. **Port 443 free** on the host (or a dedicated IP via `BIND_ADDR`). Registry URLs carry no port, so it must be 443.
4. **Port 80 free** if you cache apt (on by default). Not needed for registries. See [Caching apt packages](#caching-apt-packages) to change or switch it off.

## Quick start

```bash
cp .env.example .env
mkdir -p certs && cp /path/to/server.crt /path/to/server.key certs/   # your certificate, see below
docker compose up -d        # pulls ameer1234567890/homelab-cache from Docker Hub (also on ghcr.io)
```

Open `http://<host>:8080`. The **Connect a client** button gives commands for your actual hostnames. Then follow [Client setup](#client-setup-system-wide).

## The image

`docker-compose.yml` runs the image named by `REGISTRY_IMAGE` (default `ameer1234567890/homelab-cache:latest`). Pin a version tag such as `:1.0.0` in `.env` if you want upgrades to be deliberate, then `docker compose pull && docker compose up -d` to move to a new one. Your certificate, credentials, registry list and cached data are all supplied at run time, so nothing private is inside the image.

The same image is also published to the GitHub Container Registry. To pull from there instead, set `REGISTRY_IMAGE=ghcr.io/ameer1234567890/homelab-cache:latest` in `.env`. If the package is not public yet, make it so under the repository's Packages settings (or `docker login ghcr.io` first).

### Building it from source

```bash
docker build -t ameer1234567890/homelab-cache:latest .
docker compose up -d
```

Docker Compose only pulls an image when it is not already on the machine, so after a local build under the same name it runs your build. Run `docker compose pull` to go back to the published one. The cache binary is taken from `registry:3` by default. To use another release, such as the older 2.x line, add `--build-arg REGISTRY_BASE_IMAGE=registry:2` to the build.

### Publishing it

```bash
docker login
docker build -t ameer1234567890/homelab-cache:latest -t ameer1234567890/homelab-cache:1.0.0 .
docker push --all-tags ameer1234567890/homelab-cache
```

For both Intel/AMD and ARM machines, build multi-architecture and push in one step:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ameer1234567890/homelab-cache:latest -t ameer1234567890/homelab-cache:1.0.0 --push .
```

The GitHub Actions workflow (`.github/workflows/build.yml`) does this automatically for every pushed tag, publishing multi-architecture images to both Docker Hub and `ghcr.io/ameer1234567890/homelab-cache`. Docker Hub needs the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` repository secrets; ghcr.io uses the built-in `GITHUB_TOKEN`, so it needs no setup. You can also run the workflow by hand from the Actions tab and give it a tag name.

`.dockerignore` keeps `.env`, `credentials.env`, `certs/` and `data/` out of the build, so they cannot end up in a published layer.

## Your certificate

Put these in `./certs` (or point `CERTS_DIR` elsewhere):

| File         | What                                                                                         |
| ------------ | -------------------------------------------------------------------------------------------- |
| `server.crt` | Your certificate **with the chain** (leaf first, then intermediates and the root)            |
| `server.key` | Its private key, unencrypted PEM                                                             |
| `ca.crt`     | Optional. Your root certificate, so the UI can offer it for download to new clients          |

If both `server.crt` and `server.key` exist, they are used exactly as they are. Nothing is generated or modified, and you can mount the folder read-only (`:ro` on the `/certs` volume in `docker-compose.yml`). Different file names? Mount them individually:

```yaml
    volumes:
      - /etc/ssl/mine/fullchain.pem:/certs/server.crt:ro
      - /etc/ssl/mine/privkey.pem:/certs/server.key:ro
```

**The certificate must list every registry hostname** as a subject alternative name, because clients connect to `ghcr.io`, not to your server's name. For the default registry list:

```
registry-1.docker.io  ghcr.io  quay.io  gcr.io  registry.k8s.io  mcr.microsoft.com
```

A wildcard such as `*.microsoft.com` also covers `mcr.microsoft.com`. At start-up the container checks each name against your certificate and logs a warning for any it does not cover. You can check beforehand:

```bash
for h in registry-1.docker.io ghcr.io quay.io gcr.io registry.k8s.io mcr.microsoft.com; do
  printf '%-22s' $h; openssl x509 -in certs/server.crt -noout -checkhost $h
done
```

### No certificate yet?

Leave `certs/` empty (and writable) and the container creates a root CA (`ca.crt`, `ca.key`) and a server certificate covering every registry in your list. If you only have a root (`ca.crt` + `ca.key`), put those in and it signs the server certificate with your root. A generated certificate is reissued when the registry list or `TLS_SANS` changes, 30 days before it expires, or when `TLS_REGENERATE=true`. A `ca.crt` without `ca.key` (and no server certificate) stops the container with an explanation instead of overwriting anything.

The web UI runs as an unprivileged user and can only read a public copy of `ca.crt`, never a key.

## Choosing the registries

The list lives in `docker-compose.yml`, in the `CACHE_REGISTRIES` setting, one line per registry (`id | display name | upstream URL`). There is no separate file to maintain: if you leave the setting out, the image falls back to the same six built-in registries (Docker Hub, GHCR, Quay, GCR, registry.k8s.io, MCR).

```yaml
      CACHE_REGISTRIES: |
        hub|Docker Hub|https://registry-1.docker.io
        ghcr|GitHub Container Registry|https://ghcr.io
        # quay|Quay|https://quay.io        <- a leading # switches a line off
        internal|Our registry|https://registry.corp.example
        us-gcr|Google, US|https://us.gcr.io
```

The **hostname** in the URL is what clients type and what nginx routes on, so it must have no port and no path. The id is just a label and the data folder (`./data/<id>`). The container refuses to start, with a message naming the line, if a hostname has a port, an id is invalid, or a hostname appears twice. After editing, run `docker compose up -d`, add the new name to your client DNS, and make sure your certificate covers it.

## Client setup (system-wide)

Do this on every machine whose pulls should go through the cache: developer laptops, CI runners, Kubernetes nodes. The web UI's **Connect a client** button prints the commands for your registries.

### Quick setup with the script (Linux and macOS)

The cache serves a setup script that does steps 1 to 3 below for you. Open **Connect a client** in the web UI and copy the command shown there. It looks like this:

```bash
curl -fsSL http://CACHE-HOST:8080/setup.sh | sudo sh -s -- --fingerprint 'AA:BB:CC:...'
```

The script is generated by the cache for each request, so the address and the list of registries are always current. It:

1. downloads the root certificate and **refuses it unless its SHA-256 fingerprint matches** the one you pass,
2. adds one marked block to `/etc/hosts` pointing every registry name at the cache (the original is backed up once),
3. trusts the certificate: system-wide, and for each registry under Docker's and Podman's `certs.d`, so Docker works without a restart,
4. checks every name, reporting whether it resolves to the cache and whether TLS validates.

**Why the fingerprint matters.** This script installs a root certificate, and a plain-HTTP download can be tampered with by anyone on the network. The fingerprint in the command comes from the UI you opened, so a swapped certificate is refused and nothing is changed. Without `--fingerprint` the script asks you to confirm the fingerprint at the terminal, and stops if there is none. `--yes` skips the check, so use it only on a network you trust.

| Option                  | Effect                                                                                       |
| ----------------------- | -------------------------------------------------------------------------------------------- |
| `--dry-run`             | Show what would change and change nothing (needs no root)                                    |
| `--check`               | Only test the current setup: do the names resolve to the cache, is TLS trusted                |
| `--remove`              | Undo everything the script did: the hosts block and every certificate file it created         |
| `--no-system-trust`     | Trust the certificate for Docker and Podman only, not for the whole machine                  |
| `--no-hosts`            | Leave `/etc/hosts` alone, because your DNS already points the names at the cache             |
| `--no-trust`            | Only set up the names; use when the server certificate is issued by something already trusted |
| `--ca FILE`             | Use your own root certificate file instead of downloading one                                 |
| `--ip ADDRESS`          | Address of the cache, if the UI was reached under a name that does not resolve for the client |
| `--url URL`, `--hosts "a.io b.io"` | Override where it downloads from, or which names it sets up                       |

It is safe to run again: it replaces its own block, never duplicates it, and leaves every other line of `/etc/hosts` as it was. If a registry name is also set elsewhere in `/etc/hosts`, it warns you.

Needs root, plus `curl` or `wget`. It reads your system to choose the right trust store: Debian, Ubuntu and Alpine (`update-ca-certificates`), Red Hat, Fedora and Arch (`update-ca-trust`), SUSE, and macOS (`security`). Afterwards, restart containerd on Kubernetes nodes so it re-reads the system store.

**What it does not cover.** There is no Windows version; follow the manual steps below. The Linux behaviour is covered by automated tests, run against a sandboxed filesystem with the trust tools stubbed out. The macOS path uses Apple's documented `security` commands but has not been run on a Mac. If it misbehaves, use the manual steps.

You can also run it from a clone of this repository, which has no way to know your cache's address:

```bash
sudo sh client/setup.sh --url http://CACHE-HOST:8080 --ip CACHE-IP --hosts "registry-1.docker.io ghcr.io" --fingerprint 'AA:BB:...'
```

The manual steps follow, for other systems or if you prefer to see each one.

### 1. Send the registry names to the cache

Use the cache server's IP in place of `192.168.1.50`.

**Best for a whole network: a DNS server.** One set of records serves every client.

```ini
# dnsmasq / Pi-hole (a file in /etc/dnsmasq.d/)
address=/registry-1.docker.io/192.168.1.50
address=/ghcr.io/192.168.1.50
address=/quay.io/192.168.1.50
address=/gcr.io/192.168.1.50
address=/registry.k8s.io/192.168.1.50
address=/mcr.microsoft.com/192.168.1.50
```

```yaml
# unbound
server:
  local-data: "ghcr.io. A 192.168.1.50"
  local-data: "quay.io. A 192.168.1.50"
  # …one line per name
```

Then point the clients at that DNS server (DHCP option, router, or `resolvectl dns eth0 192.168.1.2`). Note that dnsmasq's `address=/ghcr.io/…` also covers subdomains of that name.

**Best for a few machines: the hosts file.** Add one line:

```
192.168.1.50  registry-1.docker.io ghcr.io quay.io gcr.io registry.k8s.io mcr.microsoft.com
```

- Linux and macOS: `/etc/hosts` (as root)
- Windows: `C:\Windows\System32\drivers\etc\hosts` (as Administrator)

### 2. Trust your root certificate system-wide

Download `ca.crt` from the UI (`http://<host>:8080/ca.crt`), or use your own root.

| System                  | Command                                                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Debian, Ubuntu, Alpine  | `sudo cp ca.crt /usr/local/share/ca-certificates/registry-cache.crt && sudo update-ca-certificates`                                  |
| RHEL, Fedora, Rocky     | `sudo cp ca.crt /etc/pki/ca-trust/source/anchors/registry-cache.crt && sudo update-ca-trust`                                         |
| Arch                    | `sudo trust anchor --store ca.crt`                                                                                                   |
| macOS                   | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ca.crt`                                         |
| Windows (admin PowerShell) | `Import-Certificate -FilePath .\ca.crt -CertStoreLocation Cert:\LocalMachine\Root`                                                |

If your certificate was issued by a root that machines already trust (for example your company's), skip this step.

### 3. Restart the container runtime

Runtimes read the trust store and DNS when they start:

```bash
sudo systemctl restart docker          # Docker
sudo systemctl restart containerd      # containerd, and Kubernetes nodes using it
```

Podman, Buildah and Skopeo pick the changes up immediately. On Docker Desktop (macOS, Windows) trust the root in the operating system as above and restart Docker Desktop; it normally resolves names through the host, so a hosts-file entry works there too.

**Without touching the system trust store**, Docker and Podman can trust the root per registry instead:

```bash
for h in registry-1.docker.io ghcr.io quay.io gcr.io registry.k8s.io mcr.microsoft.com; do
  sudo install -D -m 644 ca.crt /etc/docker/certs.d/$h/ca.crt          # Docker
  # sudo install -D -m 644 ca.crt /etc/containers/certs.d/$h/ca.crt    # Podman
done
```

### 4. Check it

```bash
getent hosts ghcr.io                                   # should print the cache server's IP
curl -sI https://ghcr.io/v2/ | grep -i distribution    # docker-distribution-api-version: registry/2.0
docker pull ghcr.io/<owner>/<image>:<tag>              # then look for it in the web UI
```

Kubernetes needs no `hosts.toml` or mirror settings: with DNS and trust set up on the nodes, containerd's normal pulls of `registry.k8s.io/...` and `ghcr.io/...` reach the cache.

### Clients that use an HTTP(S) proxy

This cache is a registry, not a forward proxy, so you do **not** set `HTTP_PROXY` or `HTTPS_PROXY` to point clients at it. The DNS entries above do that job.

But if the clients' container runtime is already configured to go through a corporate proxy, it would send registry traffic to that proxy, which resolves the real address and bypasses the cache. Add the registry names to `NO_PROXY` so the runtime connects to the cache directly:

```ini
# /etc/systemd/system/docker.service.d/proxy.conf   (same idea for containerd.service.d)
[Service]
Environment="HTTPS_PROXY=http://proxy.corp.example:3128"
Environment="NO_PROXY=localhost,127.0.0.1,registry-1.docker.io,ghcr.io,quay.io,gcr.io,registry.k8s.io,mcr.microsoft.com"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

For all other tools on a machine, the same variables go in `/etc/environment` (`HTTPS_PROXY=…`, `NO_PROXY=…`, plus the lower-case `https_proxy` and `no_proxy` for tools that only read those).

### If the cache itself must use a proxy

The cache's own traffic to the real registries can go through one. Set in `.env`:

```
HTTPS_PROXY=http://proxy.corp.example:3128
NO_PROXY=localhost,127.0.0.1
```

If that proxy inspects TLS, put its root certificate (PEM) in `certs/upstream/` so the cache trusts it.

### udocker on Termux (Android)

[udocker](https://github.com/indigo-dc/udocker) runs without root and has no system-wide DNS or certificate store to configure, so point its downloads at the cache through curl instead. Add these lines to `~/.curlrc`, replacing the address with your cache host's IP:

```
resolve registry-1.docker.io:443:192.168.88.11
insecure
```

`resolve` sends Docker Hub requests to the cache, and `insecure` tells curl to accept the cache's certificate without verifying it. Then pull as usual:

```bash
udocker pull alpine
```

Add one `resolve` line per registry you want cached, for example `resolve ghcr.io:443:192.168.88.11`. Note that `insecure` applies to every curl command run by that user, not only udocker, so remove the file (or the two lines) when you no longer want it.

### Undo

Remove the DNS records or hosts line, and the certificate from the trust store (delete the file you copied, then run `sudo update-ca-certificates --fresh` or `sudo update-ca-trust`). Pulls go straight to the real registries again.

## Caching apt packages

The cache also keeps the `.deb` files and package lists that `apt` downloads from Debian and Ubuntu mirrors, so a second machine (or a second `apt-get install` after a rebuild) gets them over your LAN. It works like the registries: clients keep their normal `sources.list`, and DNS sends the mirror names to the cache.

```
apt update   ──▶  deb.debian.org / archive.ubuntu.com / ...  ──▶  this container :80  ──▶  cache  ──▶  real mirror
```

Unlike registry traffic, apt files are cached by nginx itself, in `data/.apt-cache`. The folder name starts with a dot so the web UI does not mistake it for a registry.

### Choosing the mirrors

`APT_REPOSITORIES` in `docker-compose.yml` lists them, one per line: `id | display name | upstream URL`. The default is:

```
debian|Debian|https://deb.debian.org
debian-security|Debian security|https://security.debian.org
debian-ftp|Debian (ftp.debian.org)|https://ftp.debian.org
ubuntu|Ubuntu|https://archive.ubuntu.com
ubuntu-security|Ubuntu security|https://security.ubuntu.com
ubuntu-ports|Ubuntu ports (arm and others)|https://ports.ubuntu.com
docker|Docker|https://download.docker.com
tailscale|Tailscale|https://pkgs.tailscale.com
raspberrypi|Raspberry Pi OS|https://archive.raspberrypi.com
armbian|Armbian (GitHub)|https://github.armbian.com
armbian-netcup|Armbian (netcup)|https://netcup-02.armbian.com
```

The Docker, Tailscale, Raspberry Pi OS and Armbian repositories are included because many homelab machines use them. Their `sources.list` entries normally use `https://`, so clients need the cache's root certificate trusted (see below); the generated server certificate already covers these names. Delete the lines you do not need.

- **Give only the hostname.** No path and no port: clients keep their own paths (`/debian`, `/ubuntu`). The scheme says how the cache itself talks to the mirror, and `https://` is verified against the system's trusted roots plus anything in `certs/upstream`.
- **Add a line for every mirror name your clients use.** Cloud images often use regional names such as `us.archive.ubuntu.com`, or the provider's own mirror. Look in `/etc/apt/sources.list`, `/etc/apt/sources.list.d/` and `grep -r ^URIs /etc/apt/sources.list.d/` on a client. Other third-party repositories work the same way (`ppa|Launchpad PPAs|https://ppa.launchpadcontent.net`).
- Hostnames must be unique across `CACHE_REGISTRIES` and `APT_REPOSITORIES`.
- **To turn apt caching off**, comment out every line, and remove the `80:80` port line from `docker-compose.yml`.
- **If port 80 is taken on the host**, set `APT_HTTP_PORT` (for example `8081`) in `.env`. Clients then put the port in their URLs, for example `http://deb.debian.org:8081/debian`, while the name still resolves to the cache.

### Client setup

Point the mirror names at the cache, the same way as for registries: a DNS record, or a line in `/etc/hosts` on each client.

```
# dnsmasq / Pi-hole
address=/deb.debian.org/192.168.1.50
address=/security.debian.org/192.168.1.50

# or /etc/hosts
192.168.1.50  deb.debian.org security.debian.org ftp.debian.org archive.ubuntu.com security.ubuntu.com ports.ubuntu.com
192.168.1.50  download.docker.com pkgs.tailscale.com archive.raspberrypi.com github.armbian.com netcup-02.armbian.com
```

The web UI's **Connect a client** button shows these same instructions for your actual mirror names, address and port, with copy buttons. Nothing else changes: apt keeps using `http://` on port 80, which needs no certificate. Then:

```bash
sudo apt-get update && sudo apt-get install -y htop
```

> The client setup script (`setup.sh`) manages one block in `/etc/hosts` and replaces it each time it runs, so do not use it for the apt names. If you use it for the registries, add the apt names as a separate line outside its block.

**Sources that use `https://`** (Debian images that ship `https://deb.debian.org`, most third-party repositories) connect to the cache on port 443, so the client must trust the cache's root certificate (see [Client setup](#client-setup-system-wide)) and the server certificate must list the mirror name. A generated certificate already does. If you provide your own, add the mirror names to it, otherwise only `http://` sources work. The start-up log prints a note for each mirror your certificate does not cover.

### Check that it works

Ask for a small file twice. The first request is a `MISS` (fetched from the mirror), the second a `HIT`:

```bash
curl -sI http://deb.debian.org/debian/dists/stable/InRelease | grep -i x-cache-status
curl -sI http://deb.debian.org/debian/dists/stable/InRelease | grep -i x-cache-status
```

`X-Cache-Status` is on every apt response: `MISS`, `HIT`, `EXPIRED` (the mirror was asked and sent a new copy), `REVALIDATED` (the mirror was asked and the copy was still current), `STALE` (the mirror could not be reached, so the old copy was served), or `UPDATING` (a stale copy served while another request refreshes it). If it is always `MISS`, or the headers are missing, the name probably resolves to the real mirror and not to the cache: check with `getent hosts deb.debian.org`.

### If packages are not being cached

Index files (`InRelease`, `Packages.xz`) cached but no `.deb` files? Request one package file twice and look at the headers. Use any `.deb` that exists on your mirror (find one with `apt-get install --print-uris -y htop`), with the client's real mirror name:

```bash
URL=http://deb.debian.org/debian/pool/main/h/htop/htop_3.2.2-2_amd64.deb
curl -s -o /dev/null -D - $URL | grep -iE '^(HTTP|x-cache-status|x-upstream-status|location|content-length)'
curl -s -o /dev/null -D - $URL | grep -iE '^(HTTP|x-cache-status|x-upstream-status|location|content-length)'
```

The second answer should say `X-Cache-Status: HIT`. If not:

| You see | It means | What to do |
| --- | --- | --- |
| No `X-Cache-Status` header at all | The request did not reach the cache; the name resolves to the real mirror, or something else answers on that port | Check `getent hosts <mirror name>` on the client points at the cache |
| `HTTP/1.1 301`, `302`, `307` or `308` with a `Location:` header | The mirror redirects package downloads to another server, and the client then fetches from that server directly, so the cache never sees the file | Use a different mirror name for that repository, one that serves files itself |
| `X-Upstream-Status: 403`, `404`, `429` or `5xx` | The mirror refused or failed the request | Try the same URL with `curl` on a machine without the cache |
| `X-Cache-Status: MISS` both times, status `200` | The response was not stored | Look for `[error]` or `[crit]` lines from nginx in `docker compose logs homelab-cache` (for example a full disk or a permission error on `data/.apt-cache`) and report them |
| Everything above looks fine from one machine, but another machine still downloads from the real mirror | The client also has an IPv6 address for the mirror from real DNS (`/etc/hosts` and `address=` lines usually cover IPv4 only), and apt prefers it | On the client, `getent ahosts <mirror name>` must list only the cache's address. Add an IPv6 record for the cache, or return an empty answer for AAAA queries (dnsmasq: `address=/deb.debian.org/::`) |
| `HIT` on the second request | It works; packages you installed earlier came from a mirror name that is not in `APT_REPOSITORIES`, or from `https://` before the certificate was trusted | Compare the `URIs:` / `deb` lines in the client's apt sources with the mirror list |

Also check that the machine really installs through the cache: `apt-get install --print-uris -y <package>` prints the URL each package comes from. The host in those URLs must be one of the mirrors in `APT_REPOSITORIES`.

### How long files are kept

- **Package files** (`.deb`, `.udeb`, source tarballs, anything under `by-hash/`) never change once published, so they are served from the cache for `APT_PACKAGE_TTL` (default `30d`) before the mirror is asked again.
- **Everything else** (`InRelease`, `Release`, `Packages.xz`, `Sources`, `Contents`, ...) changes whenever the mirror updates, so it is served for only `APT_INDEX_TTL` (default `5m`). After that, the cache asks the mirror whether the file changed, a small request that usually returns "not modified". New security updates therefore show up within minutes.
- **Mirror down?** The cache serves what it has, even if it has expired.
- **Several clients, one new package:** only one download goes to the mirror; the others wait for it and share the result.
- Files nobody has asked for in 90 days are removed.

### Size limit

`APT_MAX_SIZE` caps the apt cache (`20GB`, `500MB`, `1.5TB`, `20GiB`, same format as `CACHE_MAX_SIZE`). Over it, nginx removes the least recently used files. Empty means no limit, which is the default. It is separate from `CACHE_MAX_SIZE`, which only counts registry images.

### What to know

- **apt has its own entries in the web UI** (see [apt in the web UI](#apt-in-the-web-ui)), but it is separate from the registries: it does not appear in "All registries", the strip of largest images or "Delete entire docker cache" (which has its own "Delete entire apt cache" button next to it), and it does not count toward `CACHE_MAX_SIZE`. Its limit is `APT_MAX_SIZE`.
- **Clear it:** `docker compose stop && rm -rf data/.apt-cache && docker compose start`.
- **Only the names in `APT_REPOSITORIES` are cached.** A mirror that is not listed still works if the client resolves it to the real address, but is not cached. If a client resolves an unlisted name to the cache, it gets the web UI instead.
- **`HTTP_PROXY` and `HTTPS_PROXY` do not apply to apt.** nginx connects to the mirrors directly, using the container's DNS (`UPSTREAM_DNS`). Registry traffic is unaffected.
- **Not a replacement for a full mirror.** It caches what your machines ask for. It does not know what a mirror has removed, so a package withdrawn upstream can be served for up to `APT_PACKAGE_TTL`.
- **Anything on these hostnames goes through the cache.** A browser that resolves `deb.debian.org` to the cache sees the mirror's pages, cached with the same rules.

## The web UI

- Total cache size, free space on the volume, and a scale drawing of the largest images.
- A list of registries with size and share of the cache, and a searchable, sortable table of images. Expand an image for its tags, platforms, sizes, build date, digest, and the `docker pull` command to copy.
- **How well it works:** bandwidth saved, hit rate and a daily chart, for the whole cache or one registry ([details below](#cache-performance)).
- **Who pulled it:** for every tag, which host requested it first and which hosts pulled it afterwards ([details below](#who-pulled-what)).
- **Why something is missing:** when a platform or layer is not cached, the reason is shown next to it ([details below](#why-some-layers-are-not-cached)).
- **Deleting:** remove one tag, a whole image, everything from one registry, the entire docker cache or the entire apt cache, each behind a warning dialog ([details below](#deleting-from-the-cache)).
- **apt mirrors:** each mirror has its own entry in the list. Select it to see the cached packages, the size on disk and the same cache performance panel ([details below](#apt-in-the-web-ui)).
- **Light and dark mode.** The toggle is saved in the browser (`localStorage`); with no saved choice the UI follows your operating system.

Sizes are read straight from the registries' storage, because the tag API of a pull-through registry asks the upstream and cannot say what is cached locally. They are compressed bytes on disk. A multi-arch tag counts only platforms that have actually been pulled. The large headline number is real disk usage, so layers shared between images count once there; per-image sizes can add up to more.

The table lists tagged images. Content that was only ever pulled by digest (`image@sha256:...`, common in Kubernetes manifests) is cached and counts toward the totals, but has no tag to show. The registry and whole-cache delete operations do remove it.

### apt in the web UI

Under **apt mirrors** in the list on the left there is one entry per mirror in `APT_REPOSITORIES`, with its size and number of cached packages. With two or more mirrors, an **All apt mirrors** entry at the top adds them all up: one list of every cached package (with a Mirror column, searchable by mirror name too), combined statistics and a single **Delete entire apt cache** button. Select a mirror, or All apt mirrors, to see:

- **The cached packages**: name, version, architecture, size and when the cache stored it, searchable and sortable. A package is a `.deb`, `.udeb` or `.ddeb` file. Everything else the mirror serves (`InRelease`, `Packages.xz`, `by-hash` files) is counted as **index files** in one line under the table, because apt refreshes those by itself.
- **Cache performance** for that mirror: bandwidth saved, hit rate and the daily chart, the same as for registries. The numbers come from nginx's own cache status for every request. A hit is any request the cache answered, including index files that were re-checked with the mirror and found unchanged. Requests for files that do not exist are not counted, and neither are `HEAD` requests.
- **The apt size limit**, if `APT_MAX_SIZE` is set: one meter for all mirrors together.
- **Deleting**: one package, or everything cached for the mirror. Statistics are kept.

How it works: nginx writes one JSON line per apt request to `data/.registry-cache/apt-access.log`, and the UI reads it the same way as the registry log and keeps its totals in `apt-stats.json` next to it. The package list comes from nginx's cache folder (`data/.apt-cache`) itself: every cached file starts with the request it answers, so nothing else has to be recorded.

- **Cached** is the time the file was stored or last re-checked with the mirror, whichever is later, not the date the mirror published the package.
- **At most 2000 packages** per mirror are listed (the newest). The count above the list is always complete.
- **Deleting is immediate.** nginx notices the missing file the next time a client asks for it and fetches it from the mirror again.
- **Only mirrors in `APT_REPOSITORIES` are listed.** Files cached for a mirror that you have since removed from the list still use disk space and count toward the limit, but they are not shown. Clear them with `rm -rf data/.apt-cache` (container stopped) if you do not need them.

### Who pulled what

Expand an image and look at **First pulled by**. It names the host whose request caused the image to be fetched and cached, and how many later pulls were served from the cache. Select it for the full history: each later host with its number of pulls, when it last pulled, and its client software (`docker/27.3.1`, `containerd/1.7.2`, ...).

How it works: nginx writes one JSON line per registry request to `data/.registry-cache/access.log`, and the UI reads it continuously and keeps its own history next to it (`tracking.json`). The raw log is emptied once it has been read and grows past 20 MB. The history survives restarts and is forgotten for an image when you delete it.

- **Hosts are client addresses**, exactly as nginx sees them. Name them in `clients.conf` (`ip|name`, one per line). It is picked up on the next scan. If a change does not show up, recreate the container (`docker compose up -d --force-recreate`): some editors replace the file, which a single-file mount does not follow.
- **Behind NAT.** Docker Desktop, another reverse proxy or a load balancer in front of the cache replaces the real client address with its own. Then every pull looks like it came from that device.
- **A pull counts once**, even though a client sends several requests for it (a check for the tag, then a request for its digest).
- **Re-cached images start over.** If an image expires and is fetched again, or its tag moves to new content, only requests since then belong to the new copy.
- **Images cached before tracking started** show "Not recorded" as the first puller. Their later pulls are listed from the moment tracking began.

### Cache performance

The **Cache performance** panel shows what the cache is worth, for the last 7 days, the last 30 days, or all time. Select a registry in the list to see only its numbers.

- **Bandwidth saved** is the data served from the cache that clients would otherwise have downloaded from the upstream registry again.
- **Hit rate** is shown by data and by number of layers. **Served from the cache** and **Fetched from upstream** are the two halves of the data. The daily chart stacks them.
- **Failed requests** counts layer downloads that failed, such as upstream rate limits. The image's own details say which layer and why.

What counts: layer downloads (layers and image configs). Manifests are a few kilobytes and are checked against upstream on every tag pull anyway, so they are left out.

**How a hit is recognised.** The registry does not say whether it served a layer from disk or fetched it, so the UI works it out. nginx logs how long each request took, which gives the moment it began. A layer whose file on disk is older than that existed beforehand, so it was a **hit**. A file written while the request ran was fetched from upstream by it, so it was a **miss**. When the file cannot settle it (a replayed log, a layer since evicted), the UI falls back to whether that layer had been served successfully before. Treat the result as a close estimate: it deliberately leans towards counting misses, so savings are understated rather than flattered.

- Two clients that start downloading an uncached layer at the same moment both count as misses, because both really did fetch it from upstream.
- A layer that expired and was fetched again counts as a miss.
- Layers cached before tracking started count as hits from their first request after it did.
- Days are UTC. Per-day numbers are kept for 90 days and lifetime totals are kept indefinitely, in `data/.registry-cache/tracking.json`. They survive restarts and deleting images, because they describe traffic rather than what is stored. To start from zero, stop the container and delete that file (which also clears the pull history).
- The panel appears only when the request log exists, so it is hidden if the UI runs outside the container.

### Why some layers are not cached

A pull-through cache stores only what clients request, so a manifest can list more than is on disk. The image still pulls normally; missing parts are fetched from the upstream registry when someone asks. When a platform or layer is missing, the table says why and **Details** lists each missing item:

| Reason shown                      | What it means                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Never requested                   | No client has asked for the layer through the cache. Clients skip layers they already have locally, or the pull stopped before reaching it. |
| Upstream fetch failed             | A request for it failed. The status is shown with the host and time, e.g. HTTP 429 (rate limit), 401/403 (add credentials), 404, 5xx, or 499 (the client cancelled mid-download). |
| Expired or removed                | It was cached earlier and is gone now: it passed `CACHE_TTL`, or was deleted from disk.                                                     |
| Cannot be cached                  | A non-distributable ("foreign") layer, such as some Windows base layers. Clients fetch it from its own source.                              |
| Downloading now                   | A download into that repository is in progress.                                                                                             |

Architectures that no client has pulled are not listed at all: for a multi-arch image that is the normal state, not a problem. A platform appears only if it is missing for a real reason, such as a failed fetch or expiry.

The failure and expiry reasons come from the request history above, so they apply from the time tracking started.

### Deleting from the cache

These operations each have their own confirmation dialog that explains what will be removed, how much space is freed, and that it cannot be undone:

| Operation                    | Where                                                        | Extra safeguard                       |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------- |
| Delete one tag               | Trash icon on a tag row                                      |                                       |
| Delete an image (all tags)   | Trash icon on the image row                                  |                                       |
| Delete everything in a registry | Select the registry, then **Delete all in ...**           | Tick "I understand" to enable Delete  |
| Delete the entire docker cache | Select **All registries**, then **Delete entire docker cache** | Tick "I understand" to enable Delete |
| Delete one apt package       | Select the mirror, then the trash icon on a package row      |                                       |
| Delete everything for an apt mirror | Select the mirror, then **Delete all in ...**         | Tick "I understand" to enable Delete  |
| Delete the entire apt cache  | Select **All registries**, any apt mirror or **All apt mirrors**, then **Delete entire apt cache** | Tick "I understand" to enable Delete |

Cancel is focused by default, so pressing Enter in the dialog never deletes.

Layers shared with other cached images are kept. Deleting `alpine:3.20` frees its own layers but not a layer that `nginx:1.27` also uses; that one is freed when the last image using it goes. A registry's data folder and your settings are never deleted.

A registry in proxy mode refuses delete requests over its API, so the UI removes the files itself and then frees every blob nothing else references. Cached copies are simply re-fetched from upstream the next time a client pulls, which **counts against upstream rate limits**. Pulls that are running at the moment of a delete may fail and need a retry.

**Controls.** Anyone who can reach the UI can delete, so:

- `UI_ADMIN_TOKEN=...` makes the UI ask for that token in the confirmation dialog.
- `UI_ALLOW_DELETE=false` removes every delete button and makes the server refuse delete requests.
- Delete requests need a custom header that a web page on another site cannot send, and requests from another origin are refused, so a malicious page cannot trigger a delete from a browser on your network.
- Put the UI behind your own authenticating reverse proxy, or bind it to a trusted interface with `BIND_ADDR`, for anything more.

**Permissions.** The registries and the UI run as an unprivileged `cache` user. On first start the container gives that user ownership of `./data` (once, recursively, so upgrading from an older version with root-owned data may take a moment on a large cache). On the host that folder then shows as owned by uid 10001. The root-only certificate key stays unreadable to them.

### Limiting the cache size

Without a limit the cache only shrinks through `CACHE_TTL`, so a busy cache can fill the disk. Set an upper size in `.env`:

```
CACHE_MAX_SIZE=50GB
```

It accepts `500MB`, `50GB`, `1.5TB` (powers of 1000, like the sizes in the UI) or `20GiB` (powers of 1024). Empty or `0` means no limit, which is the default. A value that is not a size stops the container with a message instead of being ignored. The UI then shows how much of the limit is used.

How it works:

- **Once a minute** the cache's size is checked. If it is over the limit, images are removed **least recently pulled first**, until the cache is down to **90% of the limit**. The margin stops it from removing a little every minute once the cache is full.
- **"Pulled"** means a client asked for the image, including the quick up-to-date check that `docker pull` makes when it already has the image. An image cached ten days ago and pulled two hours ago outranks one cached five days ago and never pulled since. Images with no recorded pull are ranked by when they were cached.
- **Tags on the same image count together** (`latest` and `3.20` pointing at one image): the image is in use if any of its tags is, and it goes as one.
- **Digest-only images** (`image@sha256:...`), which have no tag in the list, are removed the same way, so a pinned Kubernetes image cannot silently break the limit.
- **Shared layers** are kept while another cached image uses them, so removing an image frees only what is exclusively its own, and the clean-up keeps going until the size is really down.
- **Nothing pulled in the last five minutes is ever removed.** If that leaves the cache over the limit, it stays over until those images age, and the UI says so.
- Every removal is written to the container log (`[cache limit] removed library/nginx:1.27 ...`), and the footer of the UI shows the last clean-up.

Good to know: the size is checked every minute, so a very large pull can briefly take the cache past the limit. The size counts layer data in the cache folder, not free space on the disk, so leave headroom on the volume. `CACHE_MAX_SIZE` works independently of `UI_ALLOW_DELETE`: that setting only controls the buttons, while the limit is a server-side policy. Anything evicted is fetched from upstream again on its next pull, which counts against upstream rate limits.

## Configuration

Set in `.env` (see `.env.example`).

| Variable                       | Default            | Purpose                                                                  |
| ------------------------------ | ------------------ | ------------------------------------------------------------------------ |
| `REGISTRY_IMAGE`               | `ameer1234567890/homelab-cache:latest` | The image to run                                   |
| `BIND_ADDR`                    | `0.0.0.0`          | Interface the cache listens on (port 443)                                |
| `UI_PORT`                      | `8080`             | Plain-HTTP web UI                                                        |
| `APT_HTTP_PORT`                | `80`               | Host port for the plain-HTTP apt mirrors                                  |
| `APT_INDEX_TTL`                | `5m`               | How long apt index files are served before the mirror is asked again      |
| `APT_PACKAGE_TTL`              | `30d`              | How long `.deb` files are served before the mirror is asked again         |
| `APT_MAX_SIZE`                 | _(empty)_          | Size limit of the apt cache, such as `20GB` (empty = no limit)            |
| `UPSTREAM_DNS`, `UPSTREAM_DNS_2` | `1.1.1.1`, `8.8.8.8` | Resolvers the cache uses to reach the real registries (see warning below) |
| `HTTPS_PROXY`, `NO_PROXY`      | _(empty)_          | Outbound proxy for the cache's own upstream traffic                      |
| `CERTS_DIR`                    | `./certs`          | Your `server.crt`, `server.key`, optional `ca.crt`                       |
| `UPSTREAM_CA_DIR`              | `./certs/upstream` | Extra roots to trust when talking to upstream registries                 |
| `CACHE_DATA_DIR`               | `./data`           | Cached blobs, one sub-folder per registry                                |
| `CACHE_TTL`                    | `720h`             | Evict items this long after they were fetched (`0` = never)              |
| `CACHE_MAX_SIZE`               | _(empty)_          | Upper size limit, e.g. `50GB`; least recently pulled images go first     |
| `LOG_LEVEL`                    | `error`            | Log level of the registries (see below)                                  |
| `UI_ALLOW_DELETE`              | `true`             | `false` switches deleting from the UI off completely                     |
| `UI_ADMIN_TOKEN`               | _(empty)_          | If set, required to delete from the UI                                   |
| `TLS_SANS`                     | _(empty)_          | Extra names/IPs, only for a generated certificate                        |
| `HUB_USERNAME`, `HUB_PASSWORD` | _(empty)_          | Docker Hub login, to avoid the anonymous pull limit                      |

**Warning: `UPSTREAM_DNS` must return the real addresses.** If the cache resolved `ghcr.io` through your override DNS it would send requests to itself. The container probes for this shortly after start and logs a warning. If your network has no access to public DNS, set these to a resolver that has no overrides for the registry names.

### Docker Hub login

Docker Hub limits anonymous pulls per IP address, and every client of the cache shares the cache's address, so an unauthenticated cache can hit the limit quickly. Log it in by setting these in `.env` (or directly in the `environment:` block of `docker-compose.yml`):

```
HUB_USERNAME=your-dockerhub-username
HUB_PASSWORD=dckr_pat_xxxxxxxxxxxxxxxx
```

- **Use an access token as the password**, not your account password: Docker Hub, Account settings, Personal access tokens. Read-only access is enough, and a token can be revoked without changing your password.
- Restart to apply: `docker compose up -d`. The start-up log says `(authenticated)` next to the Docker Hub line, or `Docker Hub is anonymous` if no login is set.
- Both values are required. If only one is set, the container refuses to start and says which is missing.
- Keep `.env` private (`chmod 600 .env`). It is already in `.gitignore`.
- **Cached content is served to everyone.** The cache uses this login for every client, so anyone who can reach it can pull whatever that account can read, including its private repositories. A login also lifts the limit rather than removing it: pulls that miss the cache still count, which is where a cache helps most.
- If the limit is hit anyway, the image's missing layers show the reason ("rate-limited the request (HTTP 429)").

### Other registries

Other registries use an optional `credentials.env` (copy `credentials.env.example`). Add a pair per registry, named after its id in upper case (`-` becomes `_`):

```
GHCR_USERNAME=your-github-username
GHCR_PASSWORD=ghp_xxxxxxxxxxxxxxxx
```

The same rule applies: anyone who can reach the cache can read what these credentials can read.

### Registry log messages

The registries log at `error` by default. Registry 2.8.3 printed two messages at every start, once per registry, that are expected in cache mode and do not mean anything is wrong:

- `Registry does not implement RepositoryRemover. Will not be able to delete repos and tags`: the registry reports that a pull-through cache does not support deleting repositories and tags through its HTTP API. Nothing here uses that API. The web UI removes cached files itself, and expiry (`CACHE_TTL`) is handled inside the cache.
- `No HTTP secret provided - generated random secret`: only matters for resumable uploads, which a cache never receives. The container now supplies a secret, so this one no longer appears.

The image is now built on registry 3, which may print different start-up messages. Set `LOG_LEVEL=warn` (or `info`) in `.env` to see what yours prints, or to investigate a problem. Real failures, such as an upstream that is unreachable or rejects the login, are logged at `error` and always show.

## Maintenance

- **Clear a registry or the whole cache:** use the delete buttons in the UI. From the command line: `docker compose stop && rm -rf data/ghcr/docker && docker compose start`.
- **Reclaim space from removed manifests:** `docker compose exec homelab-cache registry garbage-collect /run/registry/hub.yml`
- **Clear the apt cache:** `docker compose stop && rm -rf data/.apt-cache && docker compose start`
- **Logs:** `docker compose logs -f homelab-cache`

## Limitations

- **apt is cached by file, not by package.** It is listed in the web UI under its own heading, apart from the registries, and has its own size limit (`APT_MAX_SIZE`). See [Caching apt packages](#caching-apt-packages).
- **A name pointed at the cache is only a registry for those clients.** Anything else at that name (for example the quay.io website in a browser) will not work from a client that resolves it to the cache. Registry API traffic (`/v2/…`) is unaffected.
- Registry names that are not in `CACHE_REGISTRIES` (for example `us.gcr.io`, `docker.elastic.co`) are not cached. Add a line for each one you use. Wildcard hosts need one entry per hostname.
- Pull-through only; pushing is not possible by design.
- One container means one failure domain: if any internal process exits, the container stops and Docker restarts all of them.
- The registry expires cached items `CACHE_TTL` after they were fetched (not after they were last used); a later pull fetches them again. Raise it, or set `0`, for a long-lived cache and watch the disk meter in the UI.
- The UI has no login of its own, and it can delete. Set `UI_ADMIN_TOKEN`, put it behind your own reverse proxy, or restrict `BIND_ADDR` on untrusted networks. Set `UI_ALLOW_DELETE=false` to make it read-only.
- Pull history needs the client's real address. See the NAT note under [Who pulled what](#who-pulled-what).
