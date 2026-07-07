# TrackMania Nations Forever / United Forever — Pelican Egg

A modern rebuild of [Hoerli1337/pterodactyl-eggs/TMNF-UF](https://github.com/Hoerli1337/pterodactyl-eggs/tree/master/TMNF-UF) for [Pelican Panel](https://pelican.dev) (`PLCN_v2` egg schema), not just a raw Pterodactyl re-export.

Two eggs are provided:

- `egg-tmnf-uf.json` — plain dedicated server, no extras.
- `egg-tmnf-uf-xaseco.json` — dedicated server + [XAseco](https://www.xaseco.org) 1.16 controller (records, Dedimania, admin commands). Requires a MySQL/MariaDB database with the XAseco table structure imported.

## What changed vs. the original egg

- **Schema**: converted from `PTDL_v2` to `PLCN_v2` (`uuid`, `tags`, `features: []`, `rules` as arrays, `sort` on variables).
- **Docker images**: both eggs now run on `ghcr.io/pelican-eggs/yolks:debian` (the `parkervcp` org that hosted the old images was renamed to `pelican-eggs`; `quay.io/parkervcp/pterodactyl-images` is gone).
- **Why the XAseco variant doesn't just use a PHP Docker image** (this went through two wrong attempts, documented here so nobody re-tries them): Pelican/Wings does not set a container's `Cmd`/`Entrypoint` at all when starting a server ([`environment/docker/container.go`](https://github.com/pelican-dev/wings/blob/main/environment/docker/container.go) `Create()`). Instead it passes the parsed startup command as a `STARTUP` environment variable ([`server/server.go`](https://github.com/pelican-dev/wings/blob/main/server/server.go) `GetEnvironmentVariables()`) and relies entirely on the **image's own baked-in entrypoint script** to read `$STARTUP` and exec it. Every `ghcr.io/pelican-eggs/yolks:*` image has that entrypoint; no other Docker image does — not `webdevops/php` (which additionally crashed Wings outright with `failed switching to "root": unable to find user root`, because it strips `root` from `/etc/passwd`), and not the official `php:*-cli` images either (which silently ignored `$STARTUP` and dropped into an interactive `php -a` shell, since their own entrypoint only knows how to run `php`). Concretely: **any egg's `docker_images` must point at a `pelican-eggs/yolks` (or equivalently-built) image, full stop** — there is no way to "just use" a generic Docker Hub image, no matter how suitable it looks otherwise.
- **No PHP yolk exists**, so the XAseco variant's install script vendors a self-contained PHP: it `apt-get install`s `php-cli`/`php-mysqli` in the (also Debian-based) installer container, then copies the `php` binary plus the full closure of its shared-library dependencies (walked recursively via `ldd`) into `/mnt/server/phproot/`. `start.sh` runs it via a small `run-php` wrapper that sets `LD_LIBRARY_PATH` and loads `mysqli.so` explicitly — fully independent of whatever the runtime image does or doesn't ship. I validated this exact vendoring technique (build the tree, then exec it from a wiped `env -i` shell with only `LD_LIBRARY_PATH` set) in my own sandbox before writing it into the egg; I could not test it inside an actual `ghcr.io/pelican-eggs/yolks:debian` Wings container, since I have no Docker access here.
- **Config now re-applies on every restart.** The original standalone egg only wrote `dedicated_cfg.txt` once, on first install (`if [ ! -f dedicated_cfg.txt ]`) — changing a variable afterwards and restarting the server did nothing. The Pelican egg now uses Pelican's built-in `config.files` XML parser, so `dedicated_cfg.txt` is rewritten from the panel's variables on every boot.
- **Fixed a real dead dependency**: the XAseco egg's `startup` script depended on `https://raw.githubusercontent.com/WhatEverest1/xaseco_egg/main/start.sh`, which now 404s. The wrapper script that boots the game server and the XAseco controller together is now generated inline by the install script — no external fetch, nothing to go missing again.
- Added the `SERVER_PASSWORD` and `Server Description` (`COMMENT`) variables to the standalone egg — the original script referenced these inside the generated config but never actually exposed them as egg variables, so they could never be set from the panel.
- Renamed the shared admin/superadmin/user XML-RPC password to a single explicit `Admin Password` / `RPC / Admin Password` variable (this mirrors what the original XAseco egg already did implicitly — Pterodactyl/Pelican's XML `find` patches every matching node, so all three `<level>` passwords were always identical either way).

## Known external dependencies you should verify before relying on this in production

I built and validated this from a sandboxed environment that cannot reach arbitrary internet hosts (only GitHub), so the following URLs — carried over from the original egg — could **not** be re-verified end-to-end:

- `http://files2.trackmaniaforever.com/TrackmaniaServer_2011-02-21.zip` — the TMF dedicated server binary. Still the URL cited in current documentation/checksums as of 2026, but it's a small third-party host.
- `https://fastdl.gamemania.org/xaseco1.16-php7.zip` — XAseco 1.16 package, from the original egg author's own CDN. If this is down, get XAseco 1.16 from [xaseco.org](https://www.xaseco.org) or the [Crytix/XASECO](https://github.com/Crytix/XASECO) GitHub mirror and swap the `XASECO_URL` in the install script.
- The tracklist still comes from this repo's `tracklist.txt` / the original repo — both work fine over GitHub.

If a download fails, the install script aborts (`set -e`) with the failing `curl` command visible in the install log, rather than silently producing a broken server.

**32-bit binary risk (XAseco variant):** `TrackmaniaServer` is a 2011-era binary and may be 32-bit. `ghcr.io/pelican-eggs/yolks:debian` is a 64-bit image; if it's missing `i386` multiarch libraries, the game server process inside `start.sh` will fail with something like `cannot execute binary file` or a missing-library error even though PHP/XAseco itself starts fine. I could not launch an actual container to verify this from my sandbox (no Docker access here). If you hit this, add `dpkg --add-architecture i386 && apt-get update && apt-get install -y libc6:i386` to the install script before the download step, or check the [Crytix/XASECO](https://github.com/Crytix/XASECO) project for a container recipe that already handles this.

## Installing

1. In Pelican admin: **Eggs → Import Egg**, upload `egg-tmnf-uf.json` (and/or `egg-tmnf-uf-xaseco.json`).
2. Create a server from the egg. Fill in your TrackMania account (`MASTERSERVER_ACCOUNT_USER`/`_PWD` or `MASTER_LOGIN`/`MASTER_PASSWORD`) and an admin password.
3. Allocate at least **2 ports**: the default allocation (game/UDP) and one more for the P2P port (`SERVER_P2P_PORT`, UDP) — put the extra allocated port number into that variable.
4. Start the server. Watch the install log for the download steps; if `curl` fails, see the note above.

### XAseco variant only

1. Create the server with a **deliberately wrong** database password first, so the install finishes without waiting on a live DB.
2. Create a MySQL/MariaDB database and update `DB_HOST` / `DB_USER` / `DB_PASSWORD` / `DB_NAME` (these are not live-editable — reinstall the server, or edit the files under `/xaseco/*.xml` by hand, after changing them).
3. Import the XAseco table structure into that database (`aseco.sql`, `extra.sql`, `rasp.sql` — get these from the XAseco 1.16 package itself, under its `sql/` folder).
4. Start the server. `start.sh` boots `TrackmaniaServer`, waits 10s for its XML-RPC port to open, then runs `xaseco/controller.php` in the foreground via the vendored `phproot/run-php`.
