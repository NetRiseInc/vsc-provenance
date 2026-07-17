# Provenance — VS Code Extension

Supply-chain security for your dependencies, powered by
[NetRise Provenance](https://provenance.netrise.io). As you open or edit a
supported manifest (Python out of the box; npm opt-in), every dependency gets an
**inline icon**, a **squiggle**, and a rich **hover** showing its risk,
advisories, and repo health. Optionally, an install-time **firewall** blocks
rejected packages before they ever touch disk.

> **How it works in one line:** the extension is just the UI — it renders results
> and runs installs. The actual analysis is done by the **`netrise` binary**,
> which the extension **downloads automatically** the first time it needs it
> (nothing to install by hand). All risk decisions come from that CLI; the
> extension never guesses.

---

## Contents

1. [Quick Start](#quick-start)
2. [Install](#install)
3. [First-run setup](#first-run-setup-2-minutes)
4. [What the icons mean](#what-the-icons-mean)
5. [Supported manifests & version resolution](#supported-manifests--version-resolution)
6. [Install-time firewall](#install-time-firewall)
7. [Settings](#settings)
8. [Policy](#policy-optional)
9. [Commands](#commands)
10. [Troubleshooting](#troubleshooting)
11. [Privacy & Telemetry](#privacy)
12. [The `netrise` binary](#the-netrise-binary)
13. [Updates](#updates)

---

## Quick Start

1. **Get the `.vsix`** from the repo's
   [**Releases**](https://github.com/NetRiseInc/vsc-provenance/releases)
   page (latest at the top; file is under **Assets**).

2. **Install it:** `code --install-extension vsc-provenance-extension-<version>.vsix --force`
   (or Extensions panel → `…` → **Install from VSIX…**), then `Ctrl+Shift+P` →
   **Developer: Reload Window**.
3. **Pick your environment** (if not prod): `Ctrl+Shift+P` → **Provenance: Switch
   Environment (Dev / Prod)**.
4. **Sign in:** `Ctrl+Shift+P` → **Provenance: Sign In** → create/copy an API key
   in the browser → paste it back in VS Code.
5. **Open a manifest** (e.g. `requirements.txt`) → icons + hovers appear. The
   first time, accept the one-click prompt to **download the `netrise` binary**.
6. **Try the firewall:** `Ctrl+Shift+P` → **Provenance: Run Install Through
   Firewall…** → type a package → watch the **Provenance Firewall** output channel.

**Requirements:** VS Code **1.67.0+**, a **Provenance API key** (step 4), and the
**`netrise` binary** (auto-downloaded; Windows / macOS Intel + Apple Silicon /
Linux x64 + arm64).

---

## Install

Grab the latest **`.vsix`** from the
[**Releases**](https://github.com/NetRiseInc/vsc-provenance/releases)
page (under **Assets**), then either:

- **VS Code UI:** Extensions panel (`Ctrl+Shift+X`) → `…` menu → **Install from
  VSIX…** → pick the file.
- **Terminal:** `code --install-extension vsc-provenance-extension-<version>.vsix --force`

Then reload: `Ctrl+Shift+P` → **Developer: Reload Window**.

> **Upgrading?** Install the newer `.vsix` with `--force` and reload — your API
> key and settings are preserved. New releases are published to the Releases page.

---

## First-run setup (≈2 minutes)

### 1. Point it at the right environment (if needed)
By default the extension talks to **`https://provenance.netrise.io`** (prod). For
**dev**: `Ctrl+Shift+P` → **Provenance: Switch Environment (Dev / Prod)** (or set
`provenance.apiUrl` manually).

> ⚠ Your API key is tied to the environment it was created in. If icons all show
> ❓/⚡, the usual cause is a key/environment mismatch.

### 2. Sign in (paste your API key)
`Ctrl+Shift+P` → **Provenance: Sign In** → the Provenance **API Keys** page opens
in your browser → create a key, copy it (shown once), paste it back into the
prompt. The key is stored in VS Code **SecretStorage** — never in settings, never
logged. Sign out any time with **Provenance: Sign Out**.

### 3. Open a manifest and see results
Open any [supported file](#supported-manifests--version-resolution) (e.g.
`requirements.txt`). Within a few seconds each dependency line shows an icon and
(for non-clean packages) a squiggle. **Hover** a dependency for the full card. The
same findings appear in the **Problems** panel (`Ctrl+Shift+M`) and via
`Ctrl+Shift+P` → **Provenance: Show Findings**.

> The first evaluation with no binary present prompts *"the netrise binary was not
> found. Download it now?"* → **Download**. See [The `netrise` binary](#the-netrise-binary).

---

## What the icons mean

| Icon | Squiggle | Meaning |
|------|----------|---------|
| 🔴 | red (Error) | **Rejected** — matches a `fail-on` policy rule |
| ✅ + blue squiggle | blue (Information) | **Advisory / repo-health note** (default) — an **indirect advisory** (a contributor or transitive dep is flagged) **or** a repo-health signal (single-maintainer / deprecated / archived). None block at install, so they're informational. Set `provenance.diagnostics.advisoryAsWarning: true` to render indirect advisories as 🟡. |
| 🟡 | yellow (Warning) | **Advisory** — a `warn-on` rule fired for a **direct** advisory (or any advisory when `advisoryAsWarning` is on). |
| ℹ️ | blue (Info) | **Allowlisted** — would trigger, but approved in `.provenance.yaml`. |
| ✅ | none | **Clean** — no advisories (hover for repo health). |
| 🚧 | — | **Unsupported** — not pinned to an exact version and no lock file to resolve from (see below). |
| ❓ | — | **Not found** — package not indexed in Provenance. |
| ⚡ | — | **Error** — API / auth / network issue. |

**Hover** any dependency for: the policy rule that fired, advisories (linked to
their Provenance page), repo health (OpenSSF Scorecard, last commit, contributors,
bus factor, breached-credential signals), and **"See in Provenance"** links.

> Prefer squiggles only? Set `provenance.showInlineIcons: false` to hide the
> inline icons and keep just the squiggles + Problems entries.

---

## Supported manifests & version resolution

The Provenance API scores a **specific published release**. But manifests often
declare **ranges** (`^2.28`, `>=2.0`, `~=4.2`) that have no single version to look
up. So the extension reads the project's **sibling lock/freeze file** to find the
**exact version that will actually be installed** and evaluates *that* — the
manifest line then gets a real ✅ / ⚠️ / 🔴 verdict, and the hover notes *"Version
`X` resolved from `<lockfile>`"*.

> **The acceptable fallback:** no lock file, a dep missing from the lock, or a
> VCS/local dep → the line stays 🚧 **unsupported** (exactly as an un-pinned dep).
> Exact pins (`requests==2.31.0`) are always evaluated directly.

### PyPI (always on)

Manifests: `requirements.txt` / `requirements-*.txt` / `requirements.in` /
`requirements-*.in`, `pyproject.toml` (plain PEP 621, Poetry, or uv), `Pipfile`.
Lock files (also render on their own): `poetry.lock`, `uv.lock`, `pylock.toml`
(PEP 751), `Pipfile.lock`, `requirements.lock`.

Range → lock resolution, **first that exists wins**:

- **`pyproject.toml`** → `poetry.lock`, then `uv.lock`, then `pylock.toml`.
- **`Pipfile`** → `Pipfile.lock` (both `[packages]` and `[dev-packages]`).
- **`requirements.txt` / `requirements-*.txt` / `requirements.in`** (plain pip) →
  the PEP 751 **`pylock.toml`** if present, else a pip-tools **compiled**
  requirements file (`requirements.lock`, or a `requirements.txt` that is fully
  `==`-pinned). Typical workflow: keep ranges in `requirements.in`, run
  `pip-compile`, and the resulting pinned file is used to resolve.

Names are matched per PEP 503 (`Flask` / `ruamel.yaml` ≡ `flask` / `ruamel-yaml`).

### npm (opt-in)

Manifests: `package.json`; lock files `package-lock.json`, `yarn.lock`,
`pnpm-lock.yaml`. **Off by default** — add `"npm"` to
`provenance.enabledEcosystems` to enable it (only against a backend that supports
npm; dev today, not prod). When enabled:

- `package.json` ranges resolve from the sibling lockfile (priority
  `package-lock.json` → `yarn.lock` → `pnpm-lock.yaml`).
- Lock files evaluate only the **direct** deps declared in `package.json` (not the
  whole transitive tree) to avoid flooding the API; the firewall still inspects
  the full tree at install time.
- The firewall routes lock files unambiguously; for a bare `package.json` it
  picks the manager from the corepack `"packageManager"` field, else the sibling
  lock file, else defaults to `npm`.

---

## Install-time firewall

The edit-time icons cover *manifests*, but a `pip install` typed in a terminal
bypasses them. The **firewall** closes that gap: it runs your install through a
local proxy that evaluates each package **before** the bytes are downloaded, so a
rejected package never lands in your environment.

1. `Ctrl+Shift+P` → **Provenance: Run Install Through Firewall…**
2. Enter what to install — e.g. `requests==2.31.0`, or `-r requirements.txt`. (If
   the cursor is on a dependency line, it's pre-filled. Also reachable from the 💡
   lightbulb on a dependency, or as a **Task**: *Terminal → Run Task… →
   Provenance: Install via firewall*.)
3. Watch the **Provenance Firewall** output channel; blocked packages get an Error
   diagnostic pinned to their manifest line and a summary toast.

The firewall always performs a **real install** of the non-blocked packages
(that's the only way to reliably catch a malicious wheel) and auto-routes to the
right tool: `pip` / `pipenv` (Pipfile) / `poetry` / `uv` (pyproject.toml /
lockfiles).

### Which Python environment it installs into (pip / uv)

The firewall installs into the environment you're working in, trying these in
order (first that works):

1. **`provenance.firewall.pythonPath`** (your explicit override), then
2. the interpreter selected in the **Python** extension (`ms-python.python`), then
3. a **`.venv/`** (or `venv/`) in the workspace root, then
4. the **`VIRTUAL_ENV`** env var, then
5. the bare `pip` on `PATH`.

`pythonPath` accepts either a `python` or a `pip` executable; we normalize both.
This controls *where* packages land, never *whether* they're inspected. (pipenv
and poetry manage their own virtualenvs, so this doesn't apply to them.)

### Where each manager installs (and how to verify)

| You typed | Manager | Where | Verify with |
|---|---|---|---|
| `attrs==1.2.3` (bare) | **pip** | the resolved interpreter (above) | `pip show attrs` |
| `attrs==1.2.3` in a **uv** project | **uv** (`uv pip install`) | the project `.venv` | `uv pip show attrs` |
| `attrs==1.2.3` in a **Poetry** project | **Poetry** (`poetry run pip install`) | the venv Poetry manages (under its cache by default, **not** a project `.venv`) | `poetry run pip show attrs` |
| `-r requirements.txt` | **pip** | the resolved interpreter | `pip list` |
| `-r pyproject.toml` / `-r uv.lock` | **uv** (`uv sync`) | the project `.venv` | `uv pip list` |
| `-r pyproject.toml` / `-r poetry.lock` | **Poetry** (`poetry install`) | the venv Poetry manages | `poetry show` |
| `-r Pipfile` | **pipenv** (`pipenv install --dev`) | the venv pipenv manages | `pipenv graph` |

> **Single package in a uv / Poetry project** installs **only** what you typed
> (via `uv pip install` / `poetry run pip install`) **without** touching
> `pyproject.toml` or the lockfile. Pass a manifest (`-r pyproject.toml` /
> `-r uv.lock` / `-r poetry.lock`) to rebuild the whole env instead. We
> deliberately avoid `uv add` / `poetry add` for a single package (they re-resolve
> the *entire* project) — run those yourself after a package is vetted if you want
> it recorded.
>
> **Poetry venv location:** by default Poetry puts the venv under its cache, not
> your project. Run `poetry env info --path` to find it, and use `poetry run pip
> show <pkg>` to confirm an install. `poetry config virtualenvs.in-project true`
> switches to a project `.venv`.

### Running the firewall directly from a terminal

If you've put `netrise` [on your terminal `PATH`](#running-netrise-from-a-terminal),
you can drive it yourself:

```bash
NETRISE_API_KEY=… netrise firewall --api-url <apiUrl> [--policy <path>] -- <install command>
```

Everything **after `--`** is the real install command the firewall wraps. The API
key goes via the **`NETRISE_API_KEY`** env var, never on the command line. Notes:

- **`--no-cache-dir`** (pip) / `--no-cache` (uv) matters — the firewall can only
  inspect packages that are actually **downloaded**. The IDE command sets this.
- **`--policy`** is optional; omit it for the binary's built-in defaults.
- Wrap any installer after `--` (`-- uv pip install --no-cache <pkg>`,
  `-- pip install -r requirements.txt`, …). Run `netrise firewall --help` for all
  flags.

---

## Settings

Open **Preferences → Settings** (`Ctrl+,`) and search **"provenance"**, or edit
`settings.json`. Every contributed setting, with its default:

| Setting | Type | Default | What it does |
|---|---|---|---|
| `provenance.apiUrl` | string | `https://provenance.netrise.io` | Provenance API base URL. Override for **dev** (`https://provenance-dev.netrise.io`), on-prem, or staging. Your API key must belong to this environment. |
| `provenance.resolveVersionsFromLockfiles` | boolean | `true` | Resolve a manifest's **ranges** to the exact version in the sibling **lock file** so ranged lines get a real ✅/⚠️/🔴 verdict instead of 🚧 unsupported. Turn **off** to evaluate only what the manifest literally says (ranges stay 🚧; exact pins unaffected). |
| `provenance.enabledEcosystems` | string[] | `["pypi"]` | Which ecosystems are active. `pypi` is always on. Add `"npm"` to enable `package.json` / `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` parsing + firewall routing — **off by default**; enable only against a backend that supports npm. |
| `provenance.policy.path` | string | `.provenance.yaml` | Workspace-relative path to the [policy file](#policy-optional). Auto-detected if present. |
| `provenance.diagnostics.advisoryAsWarning` | boolean | `false` | **Off:** indirect-only advisories render ✅ green + blue squiggle + a hover note. **On:** legacy 🟡 yellow Warning. Direct advisories and rejects are unaffected. |
| `provenance.healthHoverEnabled` | boolean | `true` | Show the repo-health card on hover for **clean** packages. Reject/advisory/error hovers always show. |
| `provenance.showInlineIcons` | boolean | `true` | Show the inline status icons before each dependency. `false` keeps only squiggles + Problems entries. |
| `provenance.telemetry.enabled` | boolean | `false` | Opt-in **anonymous** usage telemetry — see [Telemetry](#telemetry). Also gated by VS Code's global `telemetry.telemetryLevel`. No code/paths/names/URLs ever sent. |
| `provenance.experimental.allowManualApiKeyEntry` | boolean | `false` | Surface the **Set API Key Manually…** command. The recommended flow is browser-based **Sign In**; enable only for air-gapped/restricted setups. |
| `provenance.firewall.binaryPath` | string | `""` | Absolute path to an existing `netrise` binary. Empty = search `PATH`, then the managed (downloaded) copy. |
| `provenance.firewall.pythonPath` | string | `""` | Absolute path to the Python (or pip) executable the firewall installs into for **pip**/**uv**. Set = takes precedence over auto-detect. Empty = auto-detect (Python ext → workspace `.venv`/`venv` → `VIRTUAL_ENV` → bare `pip`). Controls *where* packages land, not *whether* they're inspected. |
| `provenance.firewall.policyFallback` | enum | `fail-open` | How the firewall behaves when the API is unreachable: `fail-open` (permit) or `fail-closed` (reject). Auth errors (401/403) always fail-closed. |
| `provenance.firewall.addBinaryToTerminalPath` | boolean | `false` | Add the managed `netrise` binary to the `PATH` of **VS Code's integrated terminals** so you can run `netrise …` directly. Affects only VS Code terminals (never your system `PATH`); takes effect in newly-opened terminals; applies only once the managed binary is downloaded. |

### Workspace vs. User settings

When you open **Preferences → Settings** and search "provenance", the settings
editor shows two tabs — **User** and **Workspace**:

- **User** settings apply to **every** VS Code window on your machine. Good for
  personal, machine-specific choices: your `firewall.binaryPath`,
  `firewall.pythonPath`, `showInlineIcons`, `telemetry.enabled`.
- **Workspace** settings apply to **the current project only** and are written to
  `.vscode/settings.json` in the repo — so if you commit that file, your whole
  team shares them. Good for project-wide choices: `apiUrl` (which environment the
  project targets), `policy.path`, `enabledEcosystems`, `firewall.policyFallback`.

**Precedence:** Workspace **overrides** User for that folder. A common gotcha with
**Switch Environment**: if a project pins `apiUrl` at Workspace scope (via
`.vscode/settings.json`), that value wins over your User setting — the command
detects this and writes to the same scope so the switch actually takes effect.

Example project-shared `.vscode/settings.json` (commit this in your repo):

```jsonc
{
  // Target the dev environment (use your dev API key when signing in).
  "provenance.apiUrl": "https://provenance-dev.netrise.io",
  // Policy file at the repo root (also the default).
  "provenance.policy.path": ".provenance.yaml",
  // If the firewall can't reach the API, block the install (stricter).
  "provenance.firewall.policyFallback": "fail-closed"
}
```

---

## Policy (optional)

Drop a `.provenance.yaml` in the workspace root to control what's rejected vs.
warned vs. allowlisted:

```yaml
provenance:
  fail-on:
    advisory:
      enabled: true
      relationship: direct      # direct advisories → 🔴 rejected
  warn-on:
    advisory:
      enabled: true
      relationship: indirect    # indirect advisories → 🟡 advisory
  allowlist:
    packages:
      - pkg:pypi/seleniumbase   # approve a package (versionless matches any version)
```

> All rules live under the top-level `provenance:` key. Changes are picked up on
> save and applied to both the edit-time icons and the firewall. Health signals
> (archived / single-maintainer / scorecard) show on **hover** but aren't
> policy-gateable yet. The 💡 lightbulb on a flagged dependency offers
> **"Allowlist <package>"**, which appends it here for you.

---

## Commands

`Ctrl+Shift+P`, then:

- **Provenance: Sign In** / **Sign Out**
- **Provenance: Switch Environment (Dev / Prod)**
- **Provenance: Scan Workspace** — re-evaluate every open manifest
- **Provenance: Clear Cache** — drop cached verdicts and re-scan
- **Provenance: Show Findings** — jump to any finding in the Problems panel
- **Provenance: Run Install Through Firewall…**
- **Provenance: Download Firewall Binary…**

### Output channels

`Ctrl+Shift+P` → **Output: Focus on Output View**, then pick:

- **Provenance** — the main log: activation, config, what was parsed, every CLI
  spawn, per-PURL verdicts. Start here.
- **Provenance Firewall** — one block per firewall run (`BEGIN`/`END`, full
  package-manager output, per-artifact breakdown, and a `RESULT:` summary line).
- **Provenance Telemetry** — only emits when telemetry is explicitly enabled.

---

## Troubleshooting

- **No icons / a "Sign in" note** → run **Provenance: Sign In** and paste a key.
- **Everything shows ❓ or ⚡** → wrong environment or key. Make sure
  `provenance.apiUrl` matches where your key was created, then re-sign-in.
- **A dep shows 🚧 unsupported** → it's a range with no lock file to resolve from
  (or a VCS/local dep). Add/commit a lock file, or pin an exact version. Confirm
  `provenance.resolveVersionsFromLockfiles` is on.
- **"netrise binary not found"** → accept the download prompt, or set
  `provenance.firewall.binaryPath` to an existing binary.
- **Stale icons after a change** → **Provenance: Clear Cache**.
- **Firewall run says `child-failed`** → `pip`/`uv`/`poetry` itself failed (e.g. a
  version that doesn't exist), *not* the firewall. Check its output in the
  **Provenance Firewall** channel.
- **`netrise` in the terminal is the wrong version / not found after enabling
  `addBinaryToTerminalPath`** → something shadows it. Run `Get-Command netrise`
  (PowerShell): an `Alias` in your `$PROFILE` or a different `netrise` earlier on
  `PATH` wins; our injection only affects **new** integrated terminals.

---

## Privacy

- Manifests are parsed **locally**; file contents are never uploaded.
- Only package URLs (PURLs) and repo URLs go to the Provenance API.
- The API key lives in **SecretStorage** only.
- Telemetry is **off by default** (below).

### Telemetry

Telemetry is **anonymous, opt-in, and off by default** — and today it **sends
nothing anywhere**. There is no backend and no network call: when enabled, events
are written only to a local **Provenance Telemetry** output channel so you can see
exactly what *would* be sent. It records exactly two events, both counts/booleans
only:

| Event | When | Properties |
|---|---|---|
| `extension.activated` | activation | `hasApiKey` (bool), `ecosystems` (count — never *which*), `healthHover` (bool) |
| `scan.completed` | a scan finishes | `manifests` (count — never names/paths) |

**No-PII guarantees:** (1) double-gated on `provenance.telemetry.enabled` **and**
VS Code's global `telemetry.telemetryLevel`; (2) primitives only; (3) a defensive
filter drops any string that looks like a path / email / URL / PURL / free text.

---

## The `netrise` binary

The extension shells out to a local `netrise` binary for the actual evaluation
(it's the source of truth — the extension makes no risk decisions itself). The
**first time** it needs the binary and can't find one, it prompts:

> *Provenance: the netrise binary was not found. Download it now?* → **Download**

**Download** fetches the correct prebuilt archive for your platform, **verifies
its SHA-256**, extracts it, and caches it (namespaced by version) at:

```
…/globalStorage/netrise.vsc-provenance-extension/netrise/<version>/netrise[.exe]
```

You do this once; later sessions reuse it. You can also trigger it manually with
**Provenance: Download Firewall Binary…**. The lookup order is:
`provenance.firewall.binaryPath` → system `PATH` → the managed (downloaded) copy.

**Already have `netrise`?** Point at it to skip the download:
`"provenance.firewall.binaryPath": "C:\\path\\to\\netrise.exe"`.

**Upgrades are automatic.** The cache path is namespaced by version and the
extension pins the version it expects; a new build simply re-downloads (you click
**Download** again). No stale-binary footgun.

**Reclaim disk / re-test the download** by deleting the whole `netrise/` folder
(it holds only downloaded binaries; the extension re-fetches what it needs):

```powershell
# Windows (PowerShell)
Remove-Item -Recurse -Force "$env:APPDATA\Code\User\globalStorage\netrise.vsc-provenance-extension\netrise"
```
```bash
# macOS
rm -rf "$HOME/Library/Application Support/Code/User/globalStorage/netrise.vsc-provenance-extension/netrise"
# Linux
rm -rf "$HOME/.config/Code/User/globalStorage/netrise.vsc-provenance-extension/netrise"
```

> Using **VS Code Insiders**? Replace `Code` with `Code - Insiders` in the path.

### Running `netrise` from a terminal

The managed binary is **not** on your system `PATH` by default (it's invoked by
absolute path). To run it from a terminal:

1. **Opt in (recommended):** set `provenance.firewall.addBinaryToTerminalPath:
   true`. This prepends the managed binary's dir onto the `PATH` of **VS Code's
   integrated terminals only**. Open a **new** terminal afterward. Applies only
   once the binary is downloaded.
2. **Call it by full path** (no setting), e.g.
   `& "$env:APPDATA\Code\User\globalStorage\netrise.vsc-provenance-extension\netrise\<version>\netrise.exe" --help`.
3. **Install the CLI yourself** from the
   [provenance-cli releases](https://github.com/NetRiseInc/provenance-cli/releases),
   put it on your `PATH`, and point `provenance.firewall.binaryPath` at it so the
   IDE and terminal share one binary.

### Where things live (at a glance)

| Thing | Location |
|---|---|
| The extension | VS Code's extensions dir (`~/.vscode/extensions/…`). Contains no binaries — only JS + assets. |
| The managed `netrise` binary | `…/globalStorage/netrise.vsc-provenance-extension/netrise/<version>/` |
| Your API key | VS Code **SecretStorage** (OS keychain-backed; never in settings/logs) |
| Verdict cache | VS Code workspace state (per-workspace). Cleared with **Provenance: Clear Cache**. |
| Packages the firewall installs | Your Python environment (the resolved venv / interpreter) — NOT the extension |
| Policy | `<workspace>/.provenance.yaml` (you author it; optional) |

---

## Updates

New versions are published to the
[**Releases**](https://github.com/NetRiseInc/vsc-provenance/releases) page. To upgrade, download the newer
`.vsix` and reinstall with `--force`:

```bash
code --install-extension vsc-provenance-extension-<version>.vsix --force
```

Then reload: `Ctrl+Shift+P` → **Developer: Reload Window**. Your API key and
settings are preserved across upgrades. See the
[CHANGELOG](CHANGELOG.md) for what's new in each release.

