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

### T3 Code — first run (no terminal needed)

T3 Code (`https://t3.codes`, MIT, `pingdotgg/t3code`) is an agent harness control
surface for Claude Code, Codex CLI, Cursor CLI, Grok Build, and OpenCode.

1. Install `lykhoris-t3code` from this store. First boot takes a while (up to
   ~10 minutes): the container installs a build toolchain, compiles t3's
   terminal support, and downloads both binaries. Later starts are fast.
2. Open the app's logs in the Umbrel UI (**Settings → Troubleshoot → App →
   T3 Code**, or right-click the app icon → **Troubleshoot**). Every server
   start prints a **pairing QR + pairing URL + token** (`Connection string`,
   `Token`, `Pairing URL`). Look for the block after `T3 Code server is ready`.
3. On your work machine or phone: scan the QR, or open the pairing URL, or
   paste the token at `http://umbrel.local:3773/pair`. You are paired — no
   terminal involved. Tokens are short-lived; to mint a fresh set, just
   restart the app from the Umbrel UI and re-open its logs.
4. In the T3 web UI, open **Settings → Providers**, pick the Umbrel
   environment, and enable **opencode** (already installed by the app's
   bootstrap). Authenticate with an **API key entered as an environment
   variable** on the provider instance (e.g. `ANTHROPIC_API_KEY`) — all in
   the GUI. (`opencode auth login`'s OAuth device flow remains terminal-only;
   prefer API keys for a GUI-only setup.)
5. In the T3 web UI, open **Settings → Connections** and sign in to
   **T3 Connect** for the Umbrel environment (browser OAuth click-through).
   Then sign in with the same account in the desktop/mobile app and pick the
   Umbrel environment. No port forwarding, no terminal.

State persists in the app data folder (`/data` -> `userdata/`), projects in
`/workspace`. Both survive restarts/updates.

<details>
<summary>Terminal fallback (if you prefer it)</summary>

```bash
docker exec -it lykhoris-t3code_web_1 sh
opencode --version  # should print a version; if not, check app logs for [bootstrap]
opencode auth login # OAuth device flow (terminal-only)
# pairing token without the log hunt:
t3 pair --base-dir /data
# Connect without the web UI (flags go AFTER the subcommand):
t3 connect login --base-dir /data --headless
t3 connect link --base-dir /data --headless
t3 connect status --base-dir /data
```

Restart the app once from the Umbrel UI after linking.
</details>

### T3 Code as always-on hub via T3 Connect (no port forwarding)

There is no lighter "connect-only" daemon: T3 Connect is a mode of the T3 Code
server itself, not a separate binary. The server is a single Node process +
sqlite — already light; the heavy work is your provider CLIs. So this package
is the right base even if you never open the Umbrel web UI. Enable it from
**Settings → Connections** in the T3 web UI as described above — no systemd
service needed inside the container (the compose `restart: on-failure` policy
keeps it alive).

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

**t3 updates are automated.** A weekly GitHub Action
(`.github/workflows/bump-t3.yml`, plus manual trigger under Actions) checks
`npm view t3 version` and opens a PR bumping the `T3_PIN` in
`docker-compose.yml`, manifest `version`, and `releaseNotes`. Merge the PR and
Umbrel offers the update within ~5 minutes (it never auto-applies — users
click Update).

**opencode updates itself.** The container bootstrap installs it if missing and
upgrades it when upstream is newer (version rechecked at most once a day, marker
in `/data`; best-effort — T3 always starts). No store change or Umbrel update is
involved. To pin it instead, set `OPENCODE_PIN` (e.g. `"1.0.180"`) in the
bootstrap block of `docker-compose.yml`.

Manual bump (or when automation is skipped):

1. Edit `lykhoris-t3code/docker-compose.yml` → `T3_PIN="<new-version>"`.
2. Edit `lykhoris-t3code/umbrel-app.yml` → `version: "<new-version>"` + `releaseNotes`.
3. Commit, push, then on Umbrel: Community App Stores → refresh/update the app.

Version scheme: manifest `version` is `<t3-version>[.<store-rev>]` (Umbrel offers
an update on ANY version-string change). A compose/bootstrap-only change that
users should receive bumps the store revision instead, e.g. `0.0.40` → `0.0.40.1`.

## Repo layout (required by umbrelOS)

```
umbrel-app-store.yml        # id: lykhoris
lykhoris-t3code/
  umbrel-app.yml            # id must equal folder name
  docker-compose.yml        # must define app_proxy + web
  icon.svg
```
