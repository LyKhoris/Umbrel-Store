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
  only *offers* an update when manifest `version` changes — the check is plain
  string inequality (`available.version !== installed.version`), NOT semver.
- Version scheme: `<t3-version>[.<store-rev>]`, e.g. `0.0.40`, `0.0.40.1`.
  A t3 bump sets `version` to the new t3 version (drop the suffix).
  A compose/bootstrap-only change users should receive bumps the store
  revision instead (`0.0.40` → `0.0.40.1`). A commit without ANY version
  change applies to fresh installs only, silently.
- Every user-facing change MUST also update `releaseNotes` (Umbrel's Updates
  dialog shows them).
- Keep version pins in sync: the `T3_PIN="<v>"` in `docker-compose.yml` MUST
  equal the `<t3-version>` base of manifest `version`.
- t3 bumps are automated: `.github/workflows/bump-t3.yml` (weekly + manual
  dispatch) opens a PR bumping pin + version + notes. Review and merge it;
  never commit automation output blindly.

## App: lykhoris-t3code

- Runs upstream `t3` headless (`serve --mode web`, port 3773) on
  `node:24-bookworm-slim` (multi-arch amd64+arm64).
- `HOME=/data`, server `--base-dir /data`, `NPM_CONFIG_CACHE=/data/.npm` —
  T3 state, Connect auth, and opencode auth all persist in the volume.
- The opencode harness binary is kept current by the bootstrap block in
  `docker-compose.yml`: install into `/data/bin` (in `$PATH`) if missing,
  upgrade when upstream is newer, version rechecked at most daily via the
  `/data/.opencode-version-check` marker. Best-effort throughout: T3 must
  always start even if the network/download fails (guard with `if`, never
  `set -e` around network steps). Upgrade-only, never downgrade (`sort -V`).
- opencode needs NO manifest/workflow changes — it self-updates on container
  start. Pin with `OPENCODE_PIN="x.y.z"` in the bootstrap block if a release
  ever breaks T3 compatibility.
- NEVER bundle credentials, tokens, API keys, or pairing URLs. Auth
  (`opencode auth login`, `t3 connect login/link`) is always a manual,
  on-device step documented in README.md.

## Bumping t3 to a new upstream release

1. Check latest: `npm view t3 version` and/or
   `https://github.com/pingdotgg/t3code/releases`.
2. Edit `lykhoris-t3code/docker-compose.yml` → `T3_PIN="<new-version>"`.
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
`port` integer matching `APP_PORT`, `T3_PIN` == manifest `version` base.

Shell inside compose is NOT plain shell: Compose interpolates `$VAR` /
`${VAR}` at deploy time, so every shell variable in `command:` MUST be
`$$`-escaped (`$$foo`, `$${foo:-default}`). `$(...)` and `$((...))` pass
through untouched; `${APP_DATA_DIR}` in `volumes:` stays single-`$`
(intentional interpolation). After editing the bootstrap, simulate what the
container receives and syntax-check that:

```bash
npx -y js-yaml lykhoris-t3code/docker-compose.yml 2>/dev/null | node -e 'let s=""; process.stdin.on("data",d=>s+=d).on("end",()=>{ const c=JSON.parse(s); require("fs").writeFileSync("/tmp/bootstrap-src.sh", c.services.web.command[2]); });'
node -e 'const fs=require("fs"); const s=fs.readFileSync("/tmp/bootstrap-src.sh","utf8"); fs.writeFileSync("/tmp/bootstrap.sh", s.replace(/\$\$/g,"$"));' && sh -n /tmp/bootstrap.sh && echo "BOOTSTRAP OK"
```

WARNING: do NOT inline JS regexes for `$` checks in `bash -c`/`node -e`
double-quoted strings — shell quoting silently corrupts them (this burned us
once already). Use single-quoted `grep -P` on the extracted block instead:

```bash
sed -n '/^      - |$/,/^    volumes:$/p' lykhoris-t3code/docker-compose.yml \
  | grep -nP '(?<!\$)\$(?!\$|\()' && echo "UNESCAPED \$ — FIX" || echo "ESCAPING OK"
```

## Conventions

- One logical change per commit; imperative commit messages; push to `main`.
- Repo-local git identity is fine (`LyKhoris` + noreply address).
- Keep README.md's first-run and updating runbooks accurate when packaging
  changes (exact `t3 connect ... --base-dir /data` flag order matters:
  flags go AFTER the subcommand).
