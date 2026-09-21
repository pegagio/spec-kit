# Pegagio Local Development and Fork Operations

This guide is the operating manual for developing, synchronizing, releasing, and consuming this customized Spec Kit fork. It distinguishes fast local feedback from immutable fork builds: edit and test the checkout directly, then register a versioned build with mise only after the intended `main` commit is clean.

This is Pegagio-owned documentation, intentionally outside the canonical documentation navigation. Its automation uses the `pegagio:*` mise task namespace so future upstream workflows remain separate.

> Scripts are available as Bash (`.sh`), PowerShell (`.ps1`), and Python (`.py`) variants. Interactive `specify init` prompts for one; non-interactive runs default by operating system. Pass `--script sh|ps|py` to select explicitly.

## Table of Contents

Follow the lifecycle sections in order when preparing a fork release; use the local-feedback sections while editing.

- [Fork Model](#fork-model)
- [Set Up the Checkout](#set-up-the-checkout)
- [Develop and Validate Locally](#develop-and-validate-locally)
- [Synchronize Canonical Upstream](#synchronize-canonical-upstream)
- [Build and Register a Fork Release](#build-and-register-a-fork-release)
- [Consume a Fork Release](#consume-a-fork-release)
- [Refresh a Consumer Project](#refresh-a-consumer-project)
- [Additional Development Techniques](#additional-development-techniques)
- [Troubleshooting](#troubleshooting)

## Fork Model

This repository uses `origin/main` as the installable fork product and `upstream/main` as canonical GitHub Spec Kit. The fork branch contains intentional reusable changes; it is not a mirror. Do not maintain another permanent customization branch. Use a short-lived branch only when a change benefits from isolation, then fast-forward or merge it into `main` after validation.

```text
upstream/main  ── canonical GitHub Spec Kit
      │
      └── merge deliberately ──> origin/main  ── this fork's release source
                                         │
                                         └── optional feat/*, fix/*, and chore/* branches
```

Project-specific conventions belong in independently versioned extensions, presets, or bundles whenever possible. Keep this fork for behavior that should apply to every consumer of this distribution.

## Set Up the Checkout

Configure the remotes once, then use `main` as the durable local branch:

```bash
git remote add upstream git@github.com:github/spec-kit.git
git remote set-url --push upstream no_push

git switch main
git branch --set-upstream-to=origin/main main

git config fetch.prune true
git config pull.ff only
git config rerere.enabled true
```

The disabled upstream push URL prevents accidental direct pushes. Propose broadly useful changes from this fork through the normal upstream contribution flow.

Trust and install the repository's pinned build runner once:

```bash
mise trust mise.toml
mise install python uv
```

## Develop and Validate Locally

Start from current `main`, edit the checkout, and run the CLI directly for the fastest feedback:

```bash
git switch main
git pull --ff-only origin main
uv run specify --help
uv run specify init demo-project --integration codex --ignore-agent-tools --script py
```

The `pegagio:validate` task automates the normal multi-command validation loop. It synchronizes test dependencies, runs the test suite through this checkout's virtual environment, and checks the complete diff for whitespace errors:

```bash
mise run pegagio:validate
```

Run the CI lint check when the changed surface warrants it:

```bash
uvx ruff@0.15.0 check src tests
```

Commit the intended source, documentation, and version changes to `main` only after validation. A release build never packages uncommitted work.

## Synchronize Canonical Upstream

Sync upstream as a deliberate, validated change. A merge commit records exactly when canonical work entered the fork:

```bash
git fetch upstream --prune --tags
git switch main
git pull --ff-only origin main
git log --oneline main..upstream/main
git log --oneline upstream/main..main
git merge --no-ff upstream/main -m "chore: sync upstream main"

mise run pegagio:validate
git push origin main
```

Resolve conflicts locally before validation. Do not rebase or force-push `main`; it is public, installable history. Fetch tags during each sync because a branch-only pull does not guarantee the canonical tag namespace is available locally.

## Build and Register a Fork Release

The release task turns a clean `main` commit into a wheel, provenance record, machine-local mise registration, and local annotated tag. It does not push a tag, publish to PyPI, or update a consumer project.

Every different fork build needs a unique PEP 440 version and matching tag. For example, a first fork build based on the current development version can use `1.0.10.dev0+pegagio.1` and tag `v1.0.10.dev0+pegagio.1`. Advance the suffix for every different build; never assign new contents to an existing tag or version.

Do not change the package version for ordinary edits, validation runs, or upstream synchronization. Set a new version only when the clean `main` commit is ready to become a consumable fork build. When an upstream sync changes the base version, begin that base version's fork sequence at `+pegagio.1`; for example, `1.0.11.dev0+pegagio.1` follows an upstream change from `1.0.10.dev0` to `1.0.11.dev0`.

Read the current package version with:

```bash
mise run pegagio:get-version
```

The `pegagio:next-version` task derives the next usable fork version from `pyproject.toml` and locally available `v<base>+pegagio.<n>` tags. It keeps an already prepared, untagged suffix instead of skipping it:

```bash
mise run pegagio:next-version
mise run pegagio:next-version --write
git diff -- pyproject.toml
```

Use `pegagio:set-version <version>` only when you need to choose a nonstandard version explicitly.

Set the next version, inspect the isolated change, validate it, and commit it:

```bash
mise run pegagio:set-version 1.0.10.dev0+pegagio.1
git diff -- pyproject.toml
mise run pegagio:validate
git add pyproject.toml
git commit -m "chore: prepare 1.0.10.dev0+pegagio.1"
```

Build and register that exact committed version with one task:

```bash
mise run pegagio:build-register 1.0.10.dev0+pegagio.1
```

The task refuses a dirty tree, a non-`main` branch, version/tag/artifact/mise collisions, or unexpected CLI version. On success it writes artifacts under `dist/local/<version>/`, records `provenance.json`, registers `pipx:specify-cli@<version>`, and creates a local tag. It removes only artifacts and registrations it created if a later build step fails.

Verify the result before using it:

```bash
mise where "pipx:specify-cli@1.0.10.dev0+pegagio.1"
mise exec "pipx:specify-cli@1.0.10.dev0+pegagio.1" -- specify --version
shasum -a 256 dist/local/1.0.10.dev0+pegagio.1/wheelhouse/specify_cli-1.0.10.dev0+pegagio.1-py3-none-any.whl
```

Remote publication remains explicit after this local verification:

```bash
git push origin main:main
git push origin v1.0.10.dev0+pegagio.1
```

Pushing the tag shares the source identity, not the locally built wheel or mise registration. Another machine must build and register that exact tag before it can consume the fork release.

## Consume a Fork Release

A consumer project records its selected fork version in its own `mise.toml`; it never records a checkout path, wheel, or branch name. For example:

```toml
[tools]
"pipx:specify-cli" = "1.0.10.dev0+pegagio.1"
```

From this fork checkout, the `pegagio:consume` task automates consumer setup. It verifies that the fork build is registered locally, writes the exact consumer pin, trusts the consumer's `mise.toml`, installs declared tools, and verifies the selected CLI:

```bash
mise run pegagio:consume /absolute/path/to/consumer-project 1.0.10.dev0+pegagio.1
```

Review and commit the resulting consumer `mise.toml`. In consumer scripts and CI, use mise-controlled execution so shell activation cannot select a different global executable:

```bash
mise exec -- specify --version
```

To advance one consumer, publish and register the newer fork version first, then run the same task with that version. To roll back, run it again with an earlier registered version. Other projects and worktrees remain on the version in their own checked-out `mise.toml`.

Avoid `specify self upgrade` in this workflow. It currently resolves canonical `github/spec-kit` releases, whereas mise owns fork installation and consumer selection.

## Refresh a Consumer Project

Changing a project's CLI selection does not rewrite its generated files. After consuming a new release, refresh only the applicable managed artifacts and review their diff:

```bash
specify integration upgrade <key>
specify extension update
```

Specifications, plans, constitutions, source code, and Git history are outside the normal manifest-aware upgrade path. For a Codex skills integration, generated skills are snapshots of the selected version:

```bash
specify init --here --integration codex --integration-options="--skills"
```

Commit generated files only when they are part of the consumer project's intended configuration.

## Additional Development Techniques

Use `uvx` to simulate a user flow from this checkout or a pushed branch without making it the installed consumer version:

```bash
uvx --from . specify init demo-uvx --integration copilot --ignore-agent-tools --script sh
uvx --from git+https://github.com/pegagio/spec-kit.git@feature-branch specify init demo-branch-test --script py
```

Use the integration scaffold command to create the initial Python package and test skeleton for a new built-in integration:

```bash
specify integration scaffold my-agent --type markdown
specify integration scaffold my-agent --type toml
specify integration scaffold my-agent --type yaml
specify integration scaffold my-agent --type skills
```

Hyphenated keys become Python-safe package names. The scaffold does not register the integration automatically; review the generated metadata, then add the import and `_register()` call in `src/specify_cli/integrations/__init__.py`.

When testing initialization in a disposable directory, use a temporary workspace outside every source checkout:

```bash
test_root=$(mktemp -d /tmp/specify-test.XXXXXX)
uv run specify init "$test_root/demo" --integration claude --ignore-agent-tools --script sh
```

## Troubleshooting

### The consumer version is unavailable

Confirm that the exact fork build is registered locally:

```bash
mise ls pipx:specify-cli
mise where "pipx:specify-cli@1.0.10.dev0+pegagio.1"
```

If it is absent, build and register the exact source tag on this machine. Do not substitute a canonical package with the same base version.

### `specify` resolves to an unexpected version

Compare direct and mise-controlled resolution from the consumer project:

```bash
command -v specify
specify --version
mise which specify
mise exec -- specify --version
```

Treat the mise-controlled result as authoritative. Activate mise in the shell or use `mise exec --` when the direct command differs.

### Network or TLS failures

`--skip-tls` is deprecated and has no effect. Configure the environment's certificate store or proxy instead, for example with `SSL_CERT_FILE`, `HTTPS_PROXY`, or `HTTP_PROXY`.
