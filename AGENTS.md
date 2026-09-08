# AGENTS.md — LyKhoris Umbrel Store

Community App Store for umbrelOS (`https://github.com/LyKhoris/Umbrel-Store`).
Read this file before making any change. The repo must stay publicly readable:
Umbrel clones it anonymously and the app icon is fetched over plain HTTPS.

## Layout (enforced by umbrelOS)

```
umbrel-app-store.yml        # id: lykhoris (lowercase a-z and dashes only)
<app-id>/
  umbrel-app.yml            # id MUST equal the folder name
  docker-compose.yml        # MUST define app_proxy + web services
  icon.svg                  # referenced via raw.githubusercontent.com URL
```

Rules:

- Every app id MUST start with `<store-id>-` (e.g. `lykhoris-t3code`).
- `app_proxy` env MUST be `APP_HOST: <app-id>_web_1` and `APP_PORT: <port>`,
  where `<port>` equals manifest `port` (integer 1–65535).
- Compose `version: "3.7"`. No `privileged`, no `network_mode: host`,
  no host-path mounts — only `${APP_DATA_DIR}/...` volumes.

## Update mechanics (critical — do not skip)

- `umbreld` re-pulls all registered repos roughly every 5 minutes, but Umbrel
  only *offers* an update when manifest `version` increases. A commit without
  a version bump applies to fresh installs only, silently.
- Therefore every user-facing change MUST bump `version` in `umbrel-app.yml`
  AND write `releaseNotes` (shown in Umbrel's Updates dialog).
- Keep version pins in sync: the `t3@<v>` pin in `docker-compose.yml` MUST
  equal manifest `version`.

## App: lykhoris-t3code

- Runs upstream `t3` headless (`serve --mode web`, port 3773) on
  `node:24-bookworm-slim` (multi-arch amd64+arm64).
- `HOME=/data`, server `--base-dir /data`, `NPM_CONFIG_CACHE=/data/.npm` —
  T3 state, Connect auth, and opencode auth all persist in the volume.
- The opencode harness binary is bootstrapped into `/data/bin` (in `$PATH`)
  on first boot via the official install script with
  `OPENCODE_INSTALL_DIR=/data/bin ... --no-modify-path`. Best-effort: T3 must
  always start even if the download fails (guard with `if`, never `set -e`
  around network steps).
- opencode tracks latest on fresh installs; existing installs keep their
  binary (volumes persist). Pin with `--version x.y.z` on the bootstrap line.
- NEVER bundle credentials, tokens, API keys, or pairing URLs. Auth
  (`opencode auth login`, `t3 connect login/link`) is always a manual,
  on-device step documented in README.md.

## Bumping t3 to a new upstream release

1. Check latest: `npm view t3 version` and/or
   `https://github.com/pingdotgg/t3code/releases`.
2. Edit `lykhoris-t3code/docker-compose.yml` → `t3@<new-version>`.
3. Edit `lykhoris-t3code/umbrel-app.yml` → `version: "<new-version>"`
   + `releaseNotes` describing the change.
4. Validate (below), commit, push to `main`.

## Validation (this environment has no docker and no pyyaml — use these)

```bash
for f in umbrel-app-store.yml lykhoris-t3code/umbrel-app.yml lykhoris-t3code/docker-compose.yml; do
  npx -y js-yaml "$f" > /dev/null && echo "YAML OK: $f"
done
npx -y js-yaml lykhoris-t3code/docker-compose.yml 2>/dev/null | node -e "
let s=''; process.stdin.on('data',d=>s+=d).on('end',()=>{
  const c=JSON.parse(s), cmd=c.services.web.command;
  require('fs').writeFileSync('/tmp/bootstrap.sh', cmd[2]);
});" && sh -n /tmp/bootstrap.sh && echo "SH SYNTAX OK"
```

Also verify by inspection: folder name == app id, id prefixed with store id,
`port` integer matching `APP_PORT`, `t3@` pin == manifest `version`.

## Conventions

- One logical change per commit; imperative commit messages; push to `main`.
- Repo-local git identity is fine (`LyKhoris` + noreply address).
- Keep README.md's first-run and updating runbooks accurate when packaging
  changes (exact `t3 connect ... --base-dir /data` flag order matters:
  flags go AFTER the subcommand).
