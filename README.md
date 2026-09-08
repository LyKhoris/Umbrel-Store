# LyKhoris Umbrel Store

Community App Store for umbrelOS: https://github.com/LyKhoris/Umbrel-Store

Store ID: `lykhoris` — every app ID must start with `lykhoris-`.

## Add this store to umbrelOS

1. Open **App Store** from the dock.
2. Click the **three dots (top-right) > Community App Stores**.
3. Paste:
   ```
   https://github.com/LyKhoris/Umbrel-Store
   ```
   Click **Add**, then **Open** to browse/install.

## Apps

| App | ID | Upstream version | Umbrel port |
|-----|----|------------------|-------------|
| T3 Code | `lykhoris-t3code` | `t3@0.0.40` | `3773` |

### T3 Code — first run

T3 Code (`https://t3.codes`, MIT, `pingdotgg/t3code`) is an agent harness control
surface for Claude Code, Codex CLI, Cursor CLI, Grok Build, and OpenCode.

1. Install `lykhoris-t3code` from this store.
2. Open `http://umbrel.local:3773` (or via Umbrel home screen).
3. Check logs for the pairing QR/URL if the client asks for it. First boot takes
   a few extra minutes: the container installs the `opencode` binary into the
   persisted `/data/bin` (watch for `[bootstrap]` lines in the logs).
4. Authenticate opencode **inside** the app container (binary is already there,
   auth is per-user and stays manual):

```bash
docker exec -it lykhoris-t3code_web_1 sh
opencode --version  # should print a version; if not, check app logs for [bootstrap]
opencode auth login
```

State persists in the app data folder (`/data` -> `userdata/`), projects in
`/workspace`. Both survive restarts/updates.

### T3 Code as always-on hub via T3 Connect (no port forwarding)

There is no lighter "connect-only" daemon: T3 Connect is a mode of the T3 Code
server itself, not a separate binary. The server is a single Node process +
sqlite — already light; the heavy work is your provider CLIs. So this package
is the right base even if you never open the Umbrel web UI.

Setup (Umbrel Terminal / SSH, one time):

```bash
docker exec -it lykhoris-t3code_web_1 sh
# inside the container (flags go AFTER the subcommand; --base-dir /data keeps
# auth in the persisted volume):
npx -y t3@0.0.40 connect login --base-dir /data --headless
# follow the sign-in prompts (browser link + auth code over SSH), then:
npx -y t3@0.0.40 connect link --base-dir /data --headless
npx -y t3@0.0.40 connect status --base-dir /data
exit
```

Then restart the app once from the Umbrel UI (required after linking), and on
your phone/desktop sign in to the same T3 Connect account and pick the Umbrel
environment. No systemd service needed inside the container — the compose
`restart: on-failure` policy keeps it alive.

To stop cloud exposure later: `t3 connect unlink --base-dir /data` (keeps
login) or `t3 connect logout --base-dir /data` (clears it).

Alternatives without the T3 cloud relay: direct LAN pairing
(`npx -y t3@0.0.40 pair --base-dir /data` inside the container, then scan the QR), Tailscale
(`t3 serve --tailscale-serve`, and Umbrel already has a Tailscale app), or
desktop-managed SSH launch (no Umbrel app needed at all, but the server then
lives and dies with the desktop connection — not always-on).

## Updating T3 Code

Upstream releases: `https://github.com/pingdotgg/t3code/releases` and npm `t3`
dist-tags (`latest`, `nightly`).

To bump:

1. Edit `lykhoris-t3code/docker-compose.yml` → `t3@<new-version>`.
2. Edit `lykhoris-t3code/umbrel-app.yml` → `version: "<new-version>"` + `releaseNotes`.
3. Commit, push, then on Umbrel: Community App Stores → refresh/update the app.

`opencode` tracks the latest upstream release on fresh installs (auth in `/data`
is kept, the binary re-bootstraps into `/data/bin` if missing). Existing installs
keep their binary across app updates (volumes persist) — to force a refresh,
delete `data/bin/opencode` inside the app-data folder and restart the app.
To pin it, add `--version x.y.z` to the bootstrap line in `docker-compose.yml`.

Maintainer rule: only a bump of `version` in `umbrel-app.yml` makes Umbrel offer
an update (commits without a version bump are picked up on reinstall only).
Always update `releaseNotes` too — Umbrel shows them in the Updates dialog.

## Repo layout (required by umbrelOS)

```
umbrel-app-store.yml        # id: lykhoris
lykhoris-t3code/
  umbrel-app.yml            # id must equal folder name
  docker-compose.yml        # must define app_proxy + web
  icon.svg
```
