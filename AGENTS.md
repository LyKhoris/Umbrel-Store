# AGENTS.md — LyKhoris Umbrel Store

Community App Store for umbrelOS (`https://github.com/LyKhoris/Umbrel-Store`).
Read this file before making any change. The repo must stay publicly readable:
Umbrel clones it anonymously and app icons are fetched over plain HTTPS.

## Layout (enforced by umbrelOS)

```
umbrel-app-store.yml        # id: lykhoris (lowercase a-z and dashes only)
<app-id>/
  umbrel-app.yml            # id MUST equal the folder name
  docker-compose.yml        # MUST define app_proxy + web services
  icon.svg                  # referenced via raw.githubusercontent.com URL
```

Rules:

- Every app id MUST start with `<store-id>-` (e.g. `lykhoris-my-app`).
- `app_proxy` env MUST be `APP_HOST: <app-id>_web_1` and `APP_PORT: <port>`,
  where `<port>` equals manifest `port` (integer 1–65535).
- Compose `version: "3.7"`. No `privileged`, no `network_mode: host`,
  no host-path mounts — only `${APP_DATA_DIR}/...` volumes.

## Update mechanics (critical — do not skip)

- `umbreld` re-pulls all registered repos roughly every 5 minutes, but Umbrel
  only *offers* an update when manifest `version` changes — the check is plain
  string inequality (`available.version !== installed.version`), NOT semver.
- A commit without ANY version change applies to fresh installs only, silently.
- Every user-facing change MUST bump `version` AND update `releaseNotes`
  (Umbrel's Updates dialog shows them).

## Adding an app

1. Folder `<store-id>-<name>/` containing `umbrel-app.yml`,
   `docker-compose.yml`, `icon.svg`.
2. Manifest `id` == folder name; `port` free on a default Umbrel install.
3. Base images should be multi-arch (amd64 + arm64) so Pi and x86 both work.
4. NEVER bundle credentials, tokens, API keys, or pairing URLs. Anything
   per-user stays a manual, on-device step documented in README.md.

## Validation (this environment has no docker and no pyyaml — use these)

```bash
APP=lykhoris-my-app  # the folder you touched
for f in umbrel-app-store.yml "$APP/umbrel-app.yml" "$APP/docker-compose.yml"; do
  npx -y js-yaml "$f" > /dev/null && echo "YAML OK: $f"
done
```

Shell inside compose is NOT plain shell: Compose interpolates `$VAR` /
`${VAR}` at deploy time, so every shell variable in `command:` MUST be
`$$`-escaped (`$$foo`, `$${foo:-default}`). `$(...)` and `$((...))` pass
through untouched; `${APP_DATA_DIR}` in `volumes:` stays single-`$`
(intentional interpolation). After editing any embedded shell, simulate what
the container receives and syntax-check that:

```bash
npx -y js-yaml "$APP/docker-compose.yml" 2>/dev/null | node -e 'let s=""; process.stdin.on("data",d=>s+=d).on("end",()=>{ const c=JSON.parse(s); require("fs").writeFileSync("/tmp/bootstrap-src.sh", c.services.web.command[2]); });'
node -e 'const fs=require("fs"); const s=fs.readFileSync("/tmp/bootstrap-src.sh","utf8"); fs.writeFileSync("/tmp/bootstrap.sh", s.replace(/\$\$/g,"$"));' && sh -n /tmp/bootstrap.sh && echo "BOOTSTRAP OK"
```

WARNING: do NOT inline JS regexes for `$` checks in `bash -c`/`node -e`
double-quoted strings — shell quoting silently corrupts them (this burned us
once already). Use single-quoted `grep -P` on the extracted block instead:

```bash
sed -n '/^      - |$/,/^    volumes:$/p' "$APP/docker-compose.yml" \
  | grep -nP '(?<!\$)\$(?!\$|\()' && echo "UNESCAPED \$ — FIX" || echo "ESCAPING OK"
```

Also verify by inspection: folder name == app id, id prefixed with store id,
`port` integer matching `APP_PORT`.

## Conventions

- One logical change per commit; imperative commit messages; push to `main`.
- Repo-local git identity is fine (`LyKhoris` + noreply address).
- Keep README.md's app table and runbooks accurate when packaging changes.
