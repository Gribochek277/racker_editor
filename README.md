> Sanitized mirror of Forgejo `serhii/racker_editor`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/racker_editor` | Synced: 2026-10-06T11:48:32Z

---

# Racker Editor

Desktop editor (Linux + Windows) for designing 3D-printable racks compatible with [HomeRacker](https://homeracker.org/) Core, and exporting a Print Set (STL per part, bill of materials, assembly instructions) for Cura.

Status: app skeleton, shared contracts, the local quality gate and the bundled OpenSCAD
geometry service are in place; the Frame engine, UI panels and print export follow ticket by
ticket ([`docs/tasks/`](docs/tasks/), status in [`docs/tasks/STATUS.md`](docs/tasks/STATUS.md)).

- Glossary: [`CONTEXT.md`](CONTEXT.md)
- Decisions: [`docs/adr/`](docs/adr/)
- Specification: [`docs/spec/`](docs/spec/) (start with `00-overview.md`)
- For implementing agents: [`AGENTS.md`](AGENTS.md); for the orchestrator: [`docs/agents/orchestration.md`](docs/agents/orchestration.md)

## Prerequisites

- **Node LTS** — Node 22.12 or newer (the tooling requires it; `nvm install --lts` is fine).
- **Linux (headless CI/servers only)** — `xvfb`, to give Playwright's Electron run an X display:
  `sudo apt-get install xvfb`.
- **Linux (only when building AppImages)** — `libfuse2` (`sudo apt-get install libfuse2`), for
  running the bundled OpenSCAD AppImage (see below).
- On Windows nothing extra is needed beyond Node.

The Electron binary is not downloaded by `npm ci` (Electron 43 and newer fetch it lazily on
first use). The first `npm run dev` or `npm run e2e` therefore downloads it once; to prefetch
it explicitly run `npx install-electron`.

## Setup

```bash
npm ci
npm run vendor:upstream   # vendored HomeRacker models + BOSL2 (committed; only needed after a bump)
npm run fetch:openscad    # pinned OpenSCAD CLI for this OS into vendor/openscad/<platform>/ (gitignored)
```

## Commands

| Command | What it does |
|---|---|
| `npm run dev` | Starts electron-vite in watch mode and opens the app window. |
| `npm run build` | Builds main, preload and renderer into `out/`. |
| `npm run check` | The quality gate: ESLint + `tsc --noEmit` for every tsconfig + Vitest. Must exit 0. |
| `npm run test` | Vitest unit tests only (`tests/**/*.test.ts`). |
| `npm run e2e` | `npm run build`, then the Playwright `_electron` tests in `tests/e2e/`. |
| `npm run lint` / `npm run typecheck` | ESLint / TypeScript only. |
| `npm run format` / `npm run format:check` | Prettier write / check. |
| `npm run dist` | `electron-builder` packages the current OS: a Linux AppImage, or an NSIS installer + a portable exe on Windows. See [Building releases](#building-releases). |
| `npm run smoke:packaged` | Runs the newest built AppImage with `--smoke` and checks the JSON it prints; run it after `npm run dist`. |
| `npm run vendor:upstream` | Re-copies the vendored HomeRacker models (pinned commit) and BOSL2 (pinned tag) into `vendor/`; writes `VENDOR.md`. Re-runnable, text files only. |
| `npm run fetch:openscad` | Downloads the pinned OpenSCAD snapshot for the current OS into `vendor/openscad/<platform>/` and writes `vendor/openscad/VERSION`. Idempotent; if the pinned snapshot is gone it takes the oldest newer one. |

Headless Linux (no display) runs the e2e tests under Xvfb:

```bash
xvfb-run -a npm run e2e
```

## Run in Docker / in a browser

The editor also runs as a web app: one container serves the browser build and answers the
whole `RackerApi` over HTTP, so OpenSCAD, the Part library, `settings.json` and the exported
Print Sets all live on the server. The image is multi-stage — a builder installs the
dependencies, downloads the pinned OpenSCAD CLI and builds the app and the web assets; the
runtime image carries only Node, the server, the web build, the vendored `.scad` assets and
that binary, without npm or a compiler.

```bash
docker compose up --build     # build the image and serve; `docker compose up` once it exists
npm run deploy:docker         # the same, plus: wait for `healthy` and verify every address
```

Then open <http://localhost:18080> — or `http://<this-machine-ip>:18080` from any device on the
same network, because the port is published on `0.0.0.0`. `docker compose down` stops it; the data
in the volume stays.

`npm run deploy:docker` (`scripts/deploy-docker.mjs`) is the one command to run after a build that
should be live: it rebuilds the image, recreates the container, waits for the healthcheck and then
requests `/healthz` and `/` on `127.0.0.1` **and on every non-loopback IPv4 address of the
machine**, so a deployment that is only reachable from the machine itself fails loudly. The
`docker-deploy` project skill (`.agents/skills/docker-deploy/SKILL.md`) tells an agent when to run
it and how to read its output.

- **Port** — the host port is `RACKER_PORT`, default **18080**; inside the container the server
always listens on `8080`. Port 8080 is a common collision (on this machine `llama-server` owns
it), so `RACKER_PORT=8080 docker compose up -d` restores the plain mapping, and
`RACKER_PORT=18081 npm run deploy:docker` moves the deployment. The container restarts itself
(`restart: unless-stopped`), so the editor is back after a reboot without any command.
- **Volume** — everything the user owns (the user library, the user parts, `settings.json` and
the exported Print Sets) lives in the named volume `racker-data`, mounted at `/data` inside the
container, which is the `RACKER_DATA_DIR` of the server (§3.5, §1.9). It survives
`docker compose down` (delete it with `docker compose down -v`). Its location on the host is
the volume's mount point: `docker volume inspect racker-data` prints it, by default
`/var/lib/docker/volumes/<project>_racker-data/_data`.
- **OpenSCAD** — the CLI is **downloaded at image build time** (`npm run fetch:openscad` in the
builder stage, §3.1). Building the image therefore needs network access; the running container
needs none. In a browser the geometry is rendered on the server, so no plugin or local install
is involved.
- **Health** — `GET /healthz` answers `{"ok": true}`; the image's compose file probes it, so
`docker compose ps` shows the container as `healthy` once the server is up.

## Bundled OpenSCAD

`npm run fetch:openscad` puts the pinned OpenSCAD nightly (currently `2026.09.23`, from upstream
`scadm.json`) into the gitignored `vendor/openscad/<platform>/`. At runtime the app looks for it
in this order (§3.1):

1. the `RACKER_OPENSCAD` environment variable,
2. `resourcesPath/openscad/<platform>/` (packaged app),
3. `<app path>/vendor/openscad/<platform>/` (dev checkout; on Linux `OpenSCAD.AppImage`, on
   Windows `openscad.com` inside the unpacked folder).

The status bar shows the version it found, or `OpenSCAD not found` in red with the tried paths
in a tooltip. Renders land in `<userData>/mesh-cache/<sha256>.stl` plus a `.json` with the
bounding box, so a second render of the same part is instant.

Two AppImage quirks are handled by the runner: a missing FUSE retries once with
`APPIMAGE_EXTRACT_AND_RUN=1` (install `libfuse2` for the fast path), and a host
`LD_LIBRARY_PATH` is dropped for the child process, because the AppImage bundles its own Qt and
a system path makes it die with a symbol lookup error before it prints anything.

Vendored `.scad` trees (`vendor/homeracker/`, `vendor/BOSL2/`) and the built-in catalog
(`catalog/*.json`) are committed text and are regenerated with `npm run vendor:upstream`.

## Building releases

`npm run dist` builds `out/` and then runs `electron-builder` for the current OS. The artifacts
land in `dist/` (version `0.1.0` in the examples):

| OS | Targets | Artifacts |
|---|---|---|
| Linux | AppImage | `dist/Racker Editor-0.1.0.AppImage` |
| Windows | NSIS + portable | `dist/Racker Editor Setup 0.1.0.exe`, `dist/Racker Editor 0.1.0.exe` |

The bundled OpenSCAD binary is gitignored, so fetch it first — `npm run dist` ships whatever is
in `vendor/openscad/<platform>/`:

```bash
npm ci
npm run fetch:openscad   # vendor/openscad/<platform>/ (linux: OpenSCAD.AppImage, win32: openscad.com)
npm run dist
```

`electron-builder.yml` packs `out/` and `package.json` (plus the production `node_modules`) into
`app.asar`, and these text assets and binaries as `extraResources`, next to `app.asar`:

| In the package | Comes from |
|---|---|
| `resources/openscad/<platform>/` | `vendor/openscad/<platform>/` — Linux `OpenSCAD.AppImage`, Windows `openscad.com` |
| `resources/vendor/homeracker/`, `resources/vendor/BOSL2/` | the vendored `.scad` libraries |
| `resources/scad/` | our own `.scad` sources |
| `resources/catalog/` | the built-in Part Type manifests |
| `resources/assets/` | fonts and other runtime assets |

A packaged app finds all of them under `process.resourcesPath` (§3.1, §3.2), so a machine with
nothing else installed can render.

### Smoke-testing a release

The `--smoke` flag starts the packaged app **without a window**, runs `openscadInfo()` and one
render of `hr.core.support { units: 3 }`, prints `{ "ok", "version", "bbox" }` as one line of
JSON on stdout and exits 0 (on any failure: `{ "ok": false, "error": … }` and exit 1). It is
how a release proves that its bundled OpenSCAD really runs:

```bash
# Linux: runs the newest dist/Racker Editor-*.AppImage
xvfb-run -a npm run smoke:packaged
# headless box without xvfb-run: DISPLAY=:99 node scripts/smoke-packaged.mjs

# Windows: quote the path, and write the JSON to a file — a GUI build has no console
"dist\Racker Editor 0.1.0.exe" --smoke --smoke-out %TEMP%\smoke.json
type %TEMP%\smoke.json
echo %ERRORLEVEL%
```

On Linux `scripts/smoke-packaged.mjs` checks that the JSON names a version and the bounding box
of a Support of three Base Units (15 × 45 × 15 mm), so a wrong or empty part fails the check. It
starts the AppImage twice at most: when the first run cannot mount it, it retries once with
`APPIMAGE_EXTRACT_AND_RUN=1` (§3.1). Set `RACKER_SMOKE_TIMEOUT_MS` to change the 5-minute limit.

### Windows check (by hand, once per release)

On a Windows machine with Node installed:

```powershell
npm ci
npm run fetch:openscad   # downloads the pinned OpenSCAD zip into vendor/openscad/win32/
npm run dist             # NSIS installer + portable exe into dist\
```

Then run the portable release and look at the JSON it writes:

```cmd
"dist\Racker Editor 0.1.0.exe" --smoke --smoke-out %TEMP%\smoke.json
type %TEMP%\smoke.json
echo %ERRORLEVEL%
```

Expected: `{"ok":true,"version":"2026.09.23","bbox":[15,45,15]}` and exit code `0`. A
`{"ok":false,…}` line instead means the package cannot reach its bundled OpenSCAD — the `error`
field names the paths that were tried. Also start the app normally once (double-click the
portable exe, or run the NSIS installer) to see the window and the OpenSCAD version in the
status bar.

## Environment variables

Only the test harness sets these; the app sets sensible defaults otherwise.

| Variable | Effect |
|---|---|
| `RACKER_USER_DATA` | Electron `userData` directory (settings, user library). The e2e helper gives every launch a fresh temporary directory. |
| `RACKER_E2E` | `1` enables the e2e hooks: the main process appends `?e2e=1` to the renderer URL, which installs `window.__rackerTest` (§9.8). |
| `RACKER_E2E_DIALOG_DIR` | Skips the native dialogs: Save returns `<dir>/<defaultName>`, Open returns the most recently modified file in `<dir>` matching the filters, the unsaved-changes dialog answers with `RACKER_E2E_UNSAVED_CHOICE`, and error dialogs are appended to `<dir>/error-dialog.log` (one JSON object per line). |
| `RACKER_E2E_UNSAVED_CHOICE` | Answers the Save / Don't save / Cancel dialog in e2e runs: `save`, `discard` (the default) or `cancel`; anything else cancels. |
| `RACKER_OPENSCAD` | Overrides the OpenSCAD binary (first entry of the resolution order, §3.1). |
| `APPIMAGE_EXTRACT_AND_RUN` | Set to `1` by the geometry service itself when an AppImage cannot be mounted; can also be set by hand on a machine without FUSE. |
| `RACKER_SMOKE_TIMEOUT_MS` | How long `scripts/smoke-packaged.mjs` waits for `--smoke` (default 300000). |

## Rack files

A Rack is one `*.rack.json` document (§1.2). Save writes pretty-printed JSON (two-space indent,
trailing newline) after validating it against the schema; Save As proposes `<name>.rack.json`
through the `Rack (*.rack.json)` filter, and Open parses and validates before the document is
replaced — an invalid file shows its first schema issue and leaves the open document untouched.

The title bar shows `{name} • — Racker Editor` while the document differs from the last saved or
opened snapshot, so undoing back to it clears the dot. New, Open, Quit and closing the window ask
Save / Don't save / Cancel first (`Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+Shift+S`, `Ctrl+Z`, `Ctrl+Y`
and `Ctrl+Shift+Z` work everywhere, and a focused text field keeps its own `Ctrl+Z`). The renderer
owns the document: the Electron File menu only sends commands, and the small `window.rackerApp`
bridge carries them back together with the native dialogs and the dirty flag.

## Project layout

```
src/contracts/  Shared types + zod schemas (§1). Pure TypeScript.
src/engine/     Pure domain logic. No Electron, Node, DOM, three.js or React (enforced by ESLint).
src/main/       Electron main process: window, IPC handlers in src/main/ipc/, dialogs, files.
src/preload/    contextBridge exposing exactly the RackerApi as window.racker, plus the app bridge (window.rackerApp).
src/renderer/   React shell (ui/), three.js scene (scene/), zustand store (store/).
tests/          Vitest unit tests and tests/e2e/ Playwright _electron tests.
```

Parts derived from HomeRacker by Patrick Pötz (kellerlabs/homeracker), CC BY-SA 4.0 / MIT.
