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
| _(none yet)_ | | | |

## Adding an app to this store

1. Create a folder named exactly like the app ID, e.g. `lykhoris-my-app/`.
2. Inside it, add:
   - `umbrel-app.yml` — listing metadata (name, tagline, description,
     `version`, `port`, gallery, links). See the
     [official packages](https://github.com/getumbrel/umbrel-apps) for
     field reference, or start from the
     [community store template](https://github.com/getumbrel/umbrel-community-app-store).
   - `docker-compose.yml` — Compose `3.7` services, must include `app_proxy`
     with `APP_HOST: <app-id>_web_1` and `APP_PORT` matching the manifest port.
   - `icon.svg` — referenced by URL in the manifest.
3. Validate (see AGENTS.md), commit, push. The store refreshes on Umbrel
   within ~5 minutes.

## Updating an app

Umbrel only offers an update when the manifest `version` changes (plain string
inequality — any change counts). So every user-facing change must bump
`version` and describe itself in `releaseNotes` (shown in Umbrel's Updates
dialog). A commit without a version bump reaches fresh installs only, silently.
Umbrel never auto-applies updates; users click Update themselves.

## Repo layout (required by umbrelOS)

```
umbrel-app-store.yml        # id: lykhoris
<app-id>/                   # folder name == app id == lykhoris-*
  umbrel-app.yml
  docker-compose.yml        # must define app_proxy + web
  icon.svg
```
