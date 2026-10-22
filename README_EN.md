# WorkBuddy third-party fnOS installer

A **third-party installer for WorkBuddy on fnOS (FeiNiu NAS)** — shipped as a single fpk that **contains none of WorkBuddy's binary files**. At install time it fetches the official `.deb` from Tencent's official channel and unpacks it into an fnOS app.

> [!IMPORTANT]
> This project is an independent community tool with no affiliation, endorsement, or partnership with WorkBuddy's official team. It contains no official binaries; at install time it only fetches the official deb from public official channels, unmodified and un-repackaged.

[简体中文](README.md) | English

## Download

Download the package for your architecture from [GitHub Releases](https://github.com/mclhm8182/workbuddy-fnOS-installer/releases/latest):

| Architecture | Package |
| --- | --- |
| x86_64 | `fnwb.workbuddy_1.2.6_x86.fpk` |
| ARM64 / aarch64 | `fnwb.workbuddy_1.2.6_arm.fpk` |

Verify against the `SHA256SUMS` file included in the Release; re-download if it doesn't match:

```bash
sha256sum fnwb.workbuddy_1.2.6_x86.fpk
```

The App Center shows `v1.2`; inside the app the full version `v1.2.6` is displayed. Release notes for each version are in the [GitHub Releases](https://github.com/mclhm8182/workbuddy-fnOS-installer/releases/latest) description.

## Quick start

1. In the fnOS **App Center**, click **Manual Install**.
2. Upload the fpk for your architecture, and confirm the install location is the **system partition**.
3. After install, a WorkBuddy icon appears on the desktop — click it to open.

No input is required during install. **No password is needed by default** — just click the desktop icon.

## Access & login

**No password is required by default** (`AUTH_MODE=none`), so any device on your LAN / Tailnet can use it directly.

To enable a password, change `AUTH_MODE` and restart:

```bash
sudo sh -c 'echo password > /vol1/1000/app_workbuddy/AUTH_MODE'   # enable
sudo cat /vol1/1000/app_workbuddy/ACCESS_PASS                    # view password
sudo sh -c 'echo none > /vol1/1000/app_workbuddy/AUTH_MODE'      # disable
sudo pkill -f 'app_workbuddy.*codebuddy'
sudo sh -c 'cd /vol1/1000/app_workbuddy && setsid ./start.sh > run.log 2>&1 < /dev/null &'
```

> **Security note**: without a password, any device on your LAN / Tailnet can use your WorkBuddy directly (including reading/writing files and consuming your AI quota). If your NAS is exposed to an untrusted network (port forwarding, public IP), enable the password.

**Login & history**: open the app, click login, and scan the QR with the mobile App to bind your CodeBuddy account. After binding, chat history persists with your account and **is not lost across windows / devices**. This login does not depend on a public-IP callback — it completes as long as the NAS has outbound access.

### Reverse proxy

The gateway is plain HTTP, but the page keeps **WebSocket / SSE** connections open. The proxy
must forward the `Upgrade` / `Connection` headers and disable buffering, otherwise the UI spins
forever or stays blank.

**Mounted at the domain root** (`https://wb.example.com/` → `http://192.168.2.24:8090/`):

```nginx
location / {
    proxy_pass http://192.168.2.24:8090;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_buffering off;
    proxy_read_timeout 3600s;
}
```

**Mounted under a subpath** (`https://nas.example.com/wb/` → 8090): also tell the app its
prefix, or the `/assets/...` URLs will 404 (blank page again):

```bash
sudo sh -c 'echo /wb > /vol1/1000/app_workbuddy/BASE_PATH'
# then stop and start WorkBuddy in the App Center
```

`start.sh` turns `BASE_PATH` into `CODEBUDDY_GATEWAY_BASE_PATH` (1.2.6+). Delete the file to go
back to root-path mode.

## Uninstall

Just uninstall from the App Center — by default the program data (`/vol1/1000/app_workbuddy`) is **kept** for easy re-enable. For a full cleanup (irreversible):

```bash
sudo pkill -f 'app_workbuddy.*codebuddy'
sudo rm -rf /vol1/1000/app_workbuddy
```

## Troubleshooting

**Desktop icon does nothing / keeps spinning**

```bash
ss -lntp | grep 8090                              # is the port listening?
sudo tail -40 /var/log/apps/fnwb.workbuddy.log     # app log
sudo tail -40 /vol1/1000/app_workbuddy/run.log     # service runtime log
```

> **Fixed in 1.2.6 — "API-only page / blank screen":** the WorkBuddy 5.6 deb ships the gateway API
> but no longer ships the web UI (`cli/dist/web-ui`), so the root path only returned
> "Web UI is not built". Since 1.2.6 the installer downloads the matching `dist/web-ui` from the
> official `@tencent-ai/codebuddy-code` npm package and injects it, so the full UI loads again.
> Already installed? Just reinstall once to pick it up.

> **Fixed in 1.2.5 — "installs but won't open / blank page":** since WorkBuddy 5.6 the official deb
> no longer ships the full CLI bundle (`dist/codebuddy.js`), so the old installer's bare `--serve`
> exited with `Cannot find module '../dist/codebuddy'`, leaving port 8090 unbound. Upgrade to 1.2.5;
> no config change needed.

**"Port 8090 already in use"**: stop the occupying process, or change `APP_PORT` in `cmd/lib/common.sh` and `service_port` in `manifest`.

**Download failed**: download the official deb on your computer, copy it to the NAS, set `DEB_FILE=/tmp/wb.deb`, then reinstall from the App Center (bypasses the official endpoint query).

**Service gone after reboot**: fnOS starts the app through `cmd/main start` and normally auto-starts it on boot; if not, stop and start it in the App Center, or run `sudo /usr/local/apps/@appcenter/fnwb.workbuddy/cmd/main start`.

## Architecture & size

| | x86_64 | aarch64 |
| --- | --- | --- |
| Package | `fnwb.workbuddy_1.2.6_x86.fpk` | `fnwb.workbuddy_1.2.6_arm.fpk` |
| NAS models | Most Intel / AMD fnOS devices | ARM models such as fnOS OEC |
| Unpacked size | ~950 MB | ~2.7 GB |

Check your architecture with `uname -m`: `x86_64` → x86, `aarch64` → arm. Disk needed during install is ~2.6 GB (deb included); after install the deb is deleted and steady-state usage sits in the storage pool.

### Device-specific adaptations

- **ARM models with a small `/tmp`**: on the OEC, `/tmp` is a 977 MB tmpfs too small for the ~400 MB deb. The installer stages the download on the storage pool, and its space check follows the actual install volume.
- **Low-memory models**: the V8 heap cap scales with total RAM (2 GB → 768 MB, <12 GB → 1024 MB, larger → 1536 MB). The 2 GB OEC measures ~270 MB RSS in practice, avoiding OOM.

## Path layout

| | Location | Notes |
| --- | --- | --- |
| App registration | `/usr/local/apps/@appcenter/fnwb.workbuddy/` | System partition |
| Program body | `/vol1/1000/app_workbuddy/` | Storage pool, keeps the system partition small |

`dpkg -x` unpacking does **not** register into the dpkg database, so it cannot conflict with the official deb. If you already have a manually installed WorkBuddy (in `/vol1/1000/workbuddy`), this installer leaves it untouched — the two coexist.

## Build from source

```bash
git clone https://github.com/mclhm8182/workbuddy-fnOS-installer
cd workbuddy-fnOS-installer
./build.sh
```

Artifacts land in `dist/`: `fnwb.workbuddy_1.2.6_x86.fpk`, `fnwb.workbuddy_1.2.6_arm.fpk`, `SHA256SUMS`. `build.sh` automatically fetches fnpack 1.2.3 from fnOS's official CDN.

## Compliance & license

- This repository contains none of WorkBuddy's binaries / source code / trademark graphics; at install time it only fetches the official deb, unmodified and un-repackaged.
- Icons are original neutral artwork; no official logo is used.
- Scripts are released under the [MIT License](LICENSE). Using WorkBuddy requires registering your own account and agreeing to its terms of service.

## Acknowledgements

The fpk packaging approach follows the community project [DockFN](https://github.com/lviaa/dockfn).
