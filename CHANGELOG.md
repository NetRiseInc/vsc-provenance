# Changelog

All notable changes to the **Provenance** VS Code extension are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/) and the
project uses [Semantic Versioning](https://semver.org/).

## [1.1.0] - 2026-07-17

### Added

- **Plain-pip ranges now resolve to their locked version, too.** Extends the
  lock-file resolution to `requirements.txt` / `requirements-*.txt` and the
  pip-tools source file `requirements.in` / `requirements-*.in`. When such a
  manifest declares a range (or an un-pinned name), the extension now looks for a
  sibling pinned file, **first that exists wins**:
  - **`pylock.toml`** (the new PEP 751 standard lock file), else
  - a pip-tools **compiled** requirements file — `requirements.lock`, or a
    `requirements.txt` that is *fully* `==`-pinned (the output of `pip-compile`).
  The matched exact version is evaluated so the manifest line gets a real
  ✅ / ⚠️ / 🔴 verdict, with the hover noting *"Version `X` resolved from
  `<file>`"*. Typical workflow: keep ranges in `requirements.in`, run
  `pip-compile`, and the resulting pinned file is used to resolve. Names are
  matched per PEP 503.
- **`requirements.in` / `requirements-*.in` are now recognized manifests** (edit-time
  icons + firewall routing), matching the pip-tools convention.
- **Python ranges now resolve to their locked version.** A `pyproject.toml`
  (Poetry / uv) or `Pipfile` usually declares *ranges* (`requests = "^2.28"`,
  `flask = ">=2.0"`) that the API can't score on their own — they used to render
  🚧 unsupported. The extension now reads the sibling lock file to find the
  EXACT version that will actually be installed and evaluates THAT, so the
  manifest line gets a real ✅ / ⚠️ / 🔴 verdict. The hover notes *"Version `X`
  resolved from `<lockfile>`"* so it's clear where the number came from. This
  mirrors the behavior npm already had (`package.json` → `package-lock.json` /
  `yarn.lock` / `pnpm-lock.yaml`).
  - **pyproject.toml** → `poetry.lock`, then `uv.lock` (first that exists).
  - **Pipfile** → `Pipfile.lock`.
  - Names are matched per PEP 503 (`Flask` / `ruamel.yaml` match
    `flask` / `ruamel-yaml`).
  - **`requirements.txt` (plain pip) is not yet covered** — pip has no single
    standard lock file, so this is planned for a later phase (see
    `docs/LOCKFILE_RESOLUTION_PLAN.md`). Exact pins in `requirements.txt`
    (`requests==2.31.0`) already evaluate directly, unchanged.
  - **Acceptable fallback preserved:** no lock file, a dep missing from the
    lock, or a VCS/local dep → the line stays 🚧 unsupported, exactly as before.
- **npm ecosystem support (opt-in).** Add `"npm"` to the
  `provenance.enabledEcosystems` setting to enable it. When on, the extension:
  - Parses `package.json`, `package-lock.json`, `yarn.lock`, and
    `pnpm-lock.yaml` for edit-time diagnostics + inline icons (exact-pinned
    versions get a verdict; ranges/tags render 🚧 unsupported, same as Python).
  - Routes **Run Install Through Firewall** to `npm` / `yarn` / `pnpm` based on
    the manifest (`package.json`/`package-lock.json` → npm, `yarn.lock` → yarn,
    `pnpm-lock.yaml` → pnpm).

  **Off by default.** With npm disabled (the default) the extension does nothing
  for npm/yarn/pnpm manifests — no parsing, no diagnostics, and the firewall
  refuses to route them — so it stays safe against backends (e.g. prod) that
  don't support the npm ecosystem yet.
  - **Large lockfiles evaluate only DIRECT dependencies.** A `yarn.lock` /
    `package-lock.json` / `pnpm-lock.yaml` lists the entire transitive tree
    (often thousands of packages), which used to fan out into hundreds of CLI
    batches and trip the backend rate limiter. Edit-time evaluation now
    restricts a lockfile to the direct deps declared in the sibling
    `package.json`, deduped so each `package@version` is queried once. The full
    transitive tree is still inspected at install time by the firewall.
  - **`package.json` ranges resolve to the installed version from the lockfile.**
    `package.json` declares ranges (`^4.17.0`) the API can't score, which would
    render 🚧 unsupported everywhere. Instead, the extension reads the sibling
    lockfile (priority `package-lock.json` → `yarn.lock` → `pnpm-lock.yaml`) and
    evaluates the EXACT version each direct dep resolves to — the version that
    will actually be installed — so the `package.json` line gets a real
    ✅/⚠️/🔴 verdict. yarn matching is range-aware (`name@range`) with a
    by-name fallback for a stale lock; npm/pnpm match by name. The hover notes
    "version resolved from `<lockfile>`". No lock / no match → the dep stays
    unsupported as before.
- **Edit-time supply-chain signals.** Opening a Python manifest
  (`requirements.txt` / `requirements-*.txt`, `pyproject.toml` for PEP 621 / Poetry /
  uv, `Pipfile`, and the lockfiles `Pipfile.lock` / `poetry.lock` / `uv.lock`)
  evaluates every dependency and shows an **inline icon**, a **squiggle**, and a
  rich **hover** — advisories (linked to their Provenance page), typosquat
  suggestions, repo health (OpenSSF Scorecard, archived/deprecated/single-maintainer
  signals, bus factor), and contributor-risk signals (maintainer geo/org + breached
  credentials, with masked emails). Findings also appear in the **Problems** panel
  and via **Provenance: Show Findings**.
- **Install-time firewall.** `Provenance: Run Install Through Firewall…` runs a real
  `pip`/`pipenv`/`poetry`/`uv` install through the `netrise` firewall, which inspects
  every artifact **as it downloads** and **aborts on a rejected one** — so a malicious
  wheel never lands on disk. Reachable from the Command Palette, the 💡 lightbulb on a
  dependency line, and as a bindable **Task** (`provenance-firewall`). The **Provenance
  Firewall** output channel shows a per-run `BEGIN`/`END` block with a `RESULT:` summary
  (`outcome=ok | blocked | child-failed | errored`).
- **Automatic managed-binary download.** The extension never ships a binary. When
  `netrise` isn't found (lookup order: `provenance.firewall.binaryPath` setting → system
  `PATH` → managed copy), it offers a one-click download of the correct prebuilt archive
  for your OS, **verifies its SHA-256**, and caches it under global storage namespaced by
  version (so version bumps re-download cleanly). Command: **Provenance: Download Firewall
  Binary…**. Windows / macOS (Intel + Apple Silicon) / Linux (x64 + arm64) supported.
- **Run `netrise` from VS Code's integrated terminal (opt-in).** Setting
  **`provenance.firewall.addBinaryToTerminalPath`** (default `false`) prepends the managed
  binary's directory onto the `PATH` of **VS Code's integrated terminals only** — never
  your system-wide `PATH` — so you can run `netrise …` (and `netrise firewall -- …`)
  directly. Takes effect in newly-opened terminals; applies once the binary is downloaded.
- **Installs land in the right Python environment (pip/uv).** A fallback chain
  (`provenance.firewall.pythonPath` setting → Python extension → workspace `.venv`/`venv`
  → `VIRTUAL_ENV` → bare `pip`) picks the interpreter, so the firewall installs where you
  work, not into a random global pip. The setting accepts either a `python` or a `pip`
  executable. In uv/Poetry projects, a bare package installs **only** what you typed
  (`uv pip install` / `poetry run pip install`) without touching `pyproject.toml`/the
  lockfile; passing a manifest rebuilds the whole environment.
- **Workspace policy.** A `.provenance.yaml` (all rules under a top-level `provenance:`
  key) controls what's rejected / warned / allowlisted; the 💡 lightbulb offers
  **"Allowlist <package>"** to append entries for you.
- **Resilience & performance.** Stale-while-revalidate cache paint (<200ms first paint for
  cached manifests), a stale-if-error skip-spawn latch after a 401, repo-health roll-up
  (one diagnostic when several deps share a flagged repo), and synthetic ❓ verdicts for
  deps the CLI can't evaluate.

### Changed

- The `pyproject.toml` lock-file lookup now also considers **`pylock.toml`** after
  `poetry.lock` and `uv.lock` (first that exists wins).
- **New setting `provenance.resolveVersionsFromLockfiles` (default ON).** Lets
  you turn OFF the range→lock-file version resolution described above. When off,
  the extension evaluates only what a manifest literally declares — ranges stay
  🚧 unsupported and no sibling lock file is read (exact pins are unaffected).
  Toggling it re-scans open manifests.
- **Run Install Through Firewall now detects the right npm-family manager for a
  `package.json`.** Previously `-r package.json` always ran `npm install`, which

  could run npm against a yarn/pnpm project (and mint an unwanted
  `package-lock.json`). It now picks the manager from the project itself — the
  corepack `"packageManager"` field wins if present (e.g. `"pnpm@9.1.0"` → pnpm),
  otherwise the sibling lock file decides (`pnpm-lock.yaml` → pnpm,
  `yarn.lock` → yarn, `package-lock.json` → npm), defaulting to npm when there's
  no signal. Pointing directly at a lock file (`-r yarn.lock`) still forces that
  manager. (npm-family routing remains gated behind
  `provenance.enabledEcosystems` including `"npm"`.)
- **Lowered the minimum VS Code version** from `1.120.0` to `1.67.0`. The previous requirement was far higher than anything the extension actually used and forced many users to upgrade VS Code unnecessarily. `1.67.0` is the true floor (the newest API in use is `InputBoxValidationSeverity`). Both `engines.vscode` and `@types/vscode` were aligned to this version so the compiler validates against the real minimum.
- **Indirect-only advisories render ✅ green + a blue (Information) squiggle by default**
  (not 🟡 yellow), with an explanatory hover note — the firewall is unlikely to block them
  at install. Setting **`provenance.diagnostics.advisoryAsWarning`** (default `false`)
  restores the legacy yellow Warning. Direct advisories and rejects are unaffected.
- **All risk decisions come from the `netrise` CLI** (the single source of truth) — the
  extension performs no client-side policy evaluation. `provenance.enabledEcosystems` is
  PyPI-only for now (npm/cargo return in a future release).

### Fixed

- **`pylock.toml` (PEP 751) now shows icons/squiggles like every other lock
  file.** It was recognized as a *resolution source* for sibling manifests but
  wasn't itself a parsed manifest, so opening one directly showed nothing. It now
  parses its `[[packages]]` blocks (plural key) into per-line verdicts, including
  the named `pylock.<name>.toml` variant.
- **Compiled `requirements.txt` (pip-tools, with `--hash=` lines) no longer
  flags every dependency as a parse error.** The joined `\`-continuation +
  `--hash=…` / `--global-option` options are now stripped before PEP 508
  matching, so `name==x.y.z \` … pinned lines parse correctly.
- **`requirements.lock` / `requirements-*.lock` are now recognized manifests**
  (they were resolvable as a *sibling* compiled lock but weren't parsed when
  opened directly).


_Acceptable fallback preserved: no pinned file, a dep missing from it, or a
VCS/local dep → the line stays 🚧 unsupported, exactly as before. Honors the
`provenance.resolveVersionsFromLockfiles` setting._
- **Sign In** now opens the correct API keys page (`<apiUrl>/api-keys`) instead of the non-existent `<apiUrl>/settings/api-keys`.

### Security

- The API key is stored in VS Code **SecretStorage** only (never in settings, never
  logged), and is passed to the CLI via the `NETRISE_API_KEY` env var (never on argv).
  `Provenance: Set API Key Manually…` is hidden from the palette unless
  `provenance.experimental.allowManualApiKeyEntry` is enabled.

## [1.0.2] - 2026-07-17

### Changed

- Maintenance and stability improvements.

## [1.0.1] - 2026-07-17

### Changed

- Maintenance and stability improvements.

## [1.0.0] - 2026-07-17

### Added

- **Plain-pip ranges now resolve to their locked version, too.** Extends the
  lock-file resolution to `requirements.txt` / `requirements-*.txt` and the
  pip-tools source file `requirements.in` / `requirements-*.in`. When such a
  manifest declares a range (or an un-pinned name), the extension now looks for a
  sibling pinned file, **first that exists wins**:
  - **`pylock.toml`** (the new PEP 751 standard lock file), else
  - a pip-tools **compiled** requirements file — `requirements.lock`, or a
    `requirements.txt` that is *fully* `==`-pinned (the output of `pip-compile`).
  The matched exact version is evaluated so the manifest line gets a real
  ✅ / ⚠️ / 🔴 verdict, with the hover noting *"Version `X` resolved from
  `<file>`"*. Typical workflow: keep ranges in `requirements.in`, run
  `pip-compile`, and the resulting pinned file is used to resolve. Names are
  matched per PEP 503.
- **`requirements.in` / `requirements-*.in` are now recognized manifests** (edit-time
  icons + firewall routing), matching the pip-tools convention.
- **Python ranges now resolve to their locked version.** A `pyproject.toml`
  (Poetry / uv) or `Pipfile` usually declares *ranges* (`requests = "^2.28"`,
  `flask = ">=2.0"`) that the API can't score on their own — they used to render
  🚧 unsupported. The extension now reads the sibling lock file to find the
  EXACT version that will actually be installed and evaluates THAT, so the
  manifest line gets a real ✅ / ⚠️ / 🔴 verdict. The hover notes *"Version `X`
  resolved from `<lockfile>`"* so it's clear where the number came from. This
  mirrors the behavior npm already had (`package.json` → `package-lock.json` /
  `yarn.lock` / `pnpm-lock.yaml`).
  - **pyproject.toml** → `poetry.lock`, then `uv.lock` (first that exists).
  - **Pipfile** → `Pipfile.lock`.
  - Names are matched per PEP 503 (`Flask` / `ruamel.yaml` match
    `flask` / `ruamel-yaml`).
  - **`requirements.txt` (plain pip) is not yet covered** — pip has no single
    standard lock file, so this is planned for a later phase (see
    `docs/LOCKFILE_RESOLUTION_PLAN.md`). Exact pins in `requirements.txt`
    (`requests==2.31.0`) already evaluate directly, unchanged.
  - **Acceptable fallback preserved:** no lock file, a dep missing from the
    lock, or a VCS/local dep → the line stays 🚧 unsupported, exactly as before.
- **npm ecosystem support (opt-in).** Add `"npm"` to the
  `provenance.enabledEcosystems` setting to enable it. When on, the extension:
  - Parses `package.json`, `package-lock.json`, `yarn.lock`, and
    `pnpm-lock.yaml` for edit-time diagnostics + inline icons (exact-pinned
    versions get a verdict; ranges/tags render 🚧 unsupported, same as Python).
  - Routes **Run Install Through Firewall** to `npm` / `yarn` / `pnpm` based on
    the manifest (`package.json`/`package-lock.json` → npm, `yarn.lock` → yarn,
    `pnpm-lock.yaml` → pnpm).

  **Off by default.** With npm disabled (the default) the extension does nothing
  for npm/yarn/pnpm manifests — no parsing, no diagnostics, and the firewall
  refuses to route them — so it stays safe against backends (e.g. prod) that
  don't support the npm ecosystem yet.
  - **Large lockfiles evaluate only DIRECT dependencies.** A `yarn.lock` /
    `package-lock.json` / `pnpm-lock.yaml` lists the entire transitive tree
    (often thousands of packages), which used to fan out into hundreds of CLI
    batches and trip the backend rate limiter. Edit-time evaluation now
    restricts a lockfile to the direct deps declared in the sibling
    `package.json`, deduped so each `package@version` is queried once. The full
    transitive tree is still inspected at install time by the firewall.
  - **`package.json` ranges resolve to the installed version from the lockfile.**
    `package.json` declares ranges (`^4.17.0`) the API can't score, which would
    render 🚧 unsupported everywhere. Instead, the extension reads the sibling
    lockfile (priority `package-lock.json` → `yarn.lock` → `pnpm-lock.yaml`) and
    evaluates the EXACT version each direct dep resolves to — the version that
    will actually be installed — so the `package.json` line gets a real
    ✅/⚠️/🔴 verdict. yarn matching is range-aware (`name@range`) with a
    by-name fallback for a stale lock; npm/pnpm match by name. The hover notes
    "version resolved from `<lockfile>`". No lock / no match → the dep stays
    unsupported as before.
- **Edit-time supply-chain signals.** Opening a Python manifest
  (`requirements.txt` / `requirements-*.txt`, `pyproject.toml` for PEP 621 / Poetry /
  uv, `Pipfile`, and the lockfiles `Pipfile.lock` / `poetry.lock` / `uv.lock`)
  evaluates every dependency and shows an **inline icon**, a **squiggle**, and a
  rich **hover** — advisories (linked to their Provenance page), typosquat
  suggestions, repo health (OpenSSF Scorecard, archived/deprecated/single-maintainer
  signals, bus factor), and contributor-risk signals (maintainer geo/org + breached
  credentials, with masked emails). Findings also appear in the **Problems** panel
  and via **Provenance: Show Findings**.
- **Install-time firewall.** `Provenance: Run Install Through Firewall…` runs a real
  `pip`/`pipenv`/`poetry`/`uv` install through the `netrise` firewall, which inspects
  every artifact **as it downloads** and **aborts on a rejected one** — so a malicious
  wheel never lands on disk. Reachable from the Command Palette, the 💡 lightbulb on a
  dependency line, and as a bindable **Task** (`provenance-firewall`). The **Provenance
  Firewall** output channel shows a per-run `BEGIN`/`END` block with a `RESULT:` summary
  (`outcome=ok | blocked | child-failed | errored`).
- **Automatic managed-binary download.** The extension never ships a binary. When
  `netrise` isn't found (lookup order: `provenance.firewall.binaryPath` setting → system
  `PATH` → managed copy), it offers a one-click download of the correct prebuilt archive
  for your OS, **verifies its SHA-256**, and caches it under global storage namespaced by
  version (so version bumps re-download cleanly). Command: **Provenance: Download Firewall
  Binary…**. Windows / macOS (Intel + Apple Silicon) / Linux (x64 + arm64) supported.
- **Run `netrise` from VS Code's integrated terminal (opt-in).** Setting
  **`provenance.firewall.addBinaryToTerminalPath`** (default `false`) prepends the managed
  binary's directory onto the `PATH` of **VS Code's integrated terminals only** — never
  your system-wide `PATH` — so you can run `netrise …` (and `netrise firewall -- …`)
  directly. Takes effect in newly-opened terminals; applies once the binary is downloaded.
- **Installs land in the right Python environment (pip/uv).** A fallback chain
  (`provenance.firewall.pythonPath` setting → Python extension → workspace `.venv`/`venv`
  → `VIRTUAL_ENV` → bare `pip`) picks the interpreter, so the firewall installs where you
  work, not into a random global pip. The setting accepts either a `python` or a `pip`
  executable. In uv/Poetry projects, a bare package installs **only** what you typed
  (`uv pip install` / `poetry run pip install`) without touching `pyproject.toml`/the
  lockfile; passing a manifest rebuilds the whole environment.
- **Workspace policy.** A `.provenance.yaml` (all rules under a top-level `provenance:`
  key) controls what's rejected / warned / allowlisted; the 💡 lightbulb offers
  **"Allowlist <package>"** to append entries for you.
- **Resilience & performance.** Stale-while-revalidate cache paint (<200ms first paint for
  cached manifests), a stale-if-error skip-spawn latch after a 401, repo-health roll-up
  (one diagnostic when several deps share a flagged repo), and synthetic ❓ verdicts for
  deps the CLI can't evaluate.

### Changed

- The `pyproject.toml` lock-file lookup now also considers **`pylock.toml`** after
  `poetry.lock` and `uv.lock` (first that exists wins).
- **New setting `provenance.resolveVersionsFromLockfiles` (default ON).** Lets
  you turn OFF the range→lock-file version resolution described above. When off,
  the extension evaluates only what a manifest literally declares — ranges stay
  🚧 unsupported and no sibling lock file is read (exact pins are unaffected).
  Toggling it re-scans open manifests.
- **Run Install Through Firewall now detects the right npm-family manager for a
  `package.json`.** Previously `-r package.json` always ran `npm install`, which

  could run npm against a yarn/pnpm project (and mint an unwanted
  `package-lock.json`). It now picks the manager from the project itself — the
  corepack `"packageManager"` field wins if present (e.g. `"pnpm@9.1.0"` → pnpm),
  otherwise the sibling lock file decides (`pnpm-lock.yaml` → pnpm,
  `yarn.lock` → yarn, `package-lock.json` → npm), defaulting to npm when there's
  no signal. Pointing directly at a lock file (`-r yarn.lock`) still forces that
  manager. (npm-family routing remains gated behind
  `provenance.enabledEcosystems` including `"npm"`.)
- **Lowered the minimum VS Code version** from `1.120.0` to `1.67.0`. The previous requirement was far higher than anything the extension actually used and forced many users to upgrade VS Code unnecessarily. `1.67.0` is the true floor (the newest API in use is `InputBoxValidationSeverity`). Both `engines.vscode` and `@types/vscode` were aligned to this version so the compiler validates against the real minimum.
- **Indirect-only advisories render ✅ green + a blue (Information) squiggle by default**
  (not 🟡 yellow), with an explanatory hover note — the firewall is unlikely to block them
  at install. Setting **`provenance.diagnostics.advisoryAsWarning`** (default `false`)
  restores the legacy yellow Warning. Direct advisories and rejects are unaffected.
- **All risk decisions come from the `netrise` CLI** (the single source of truth) — the
  extension performs no client-side policy evaluation. `provenance.enabledEcosystems` is
  PyPI-only for now (npm/cargo return in a future release).

### Fixed

- **`pylock.toml` (PEP 751) now shows icons/squiggles like every other lock
  file.** It was recognized as a *resolution source* for sibling manifests but
  wasn't itself a parsed manifest, so opening one directly showed nothing. It now
  parses its `[[packages]]` blocks (plural key) into per-line verdicts, including
  the named `pylock.<name>.toml` variant.
- **Compiled `requirements.txt` (pip-tools, with `--hash=` lines) no longer
  flags every dependency as a parse error.** The joined `\`-continuation +
  `--hash=…` / `--global-option` options are now stripped before PEP 508
  matching, so `name==x.y.z \` … pinned lines parse correctly.
- **`requirements.lock` / `requirements-*.lock` are now recognized manifests**
  (they were resolvable as a *sibling* compiled lock but weren't parsed when
  opened directly).


_Acceptable fallback preserved: no pinned file, a dep missing from it, or a
VCS/local dep → the line stays 🚧 unsupported, exactly as before. Honors the
`provenance.resolveVersionsFromLockfiles` setting._
- **Sign In** now opens the correct API keys page (`<apiUrl>/api-keys`) instead of the non-existent `<apiUrl>/settings/api-keys`.

### Security

- The API key is stored in VS Code **SecretStorage** only (never in settings, never
  logged), and is passed to the CLI via the `NETRISE_API_KEY` env var (never on argv).
  `Provenance: Set API Key Manually…` is hidden from the palette unless
  `provenance.experimental.allowManualApiKeyEntry` is enabled.
