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
- [Consume a Fork Release in Other Projects](#consume-a-fork-release-in-other-projects)
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

Start from current `main`, edit the checkout, and use the local dogfooding task for the fastest feedback:

```bash
git switch main
git pull --ff-only origin main
mise run pegagio:dogfood --help
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

### Dogfood This Checkout

Dogfooding is intentionally limited to this Spec Kit checkout. The `pegagio:dogfood` task runs this repository's editable CLI and verifies that its reported version matches `pyproject.toml`, so it cannot silently run stale package metadata.

Run it from this checkout with any Specify arguments:

```bash
mise run pegagio:dogfood --version
```

The editable install reads ordinary Python source, templates, and bundled-script changes immediately. Run `uv sync --extra test` only after changing dependencies, `pyproject.toml`, console-entry-point/package configuration, or recreating `.venv`.

Its reported version should end in `+pegagio.dev`, which identifies it as live source rather than a registered build. Do not link this editable environment into another project. Other projects use only a registered, pinned release.

### One-Time Local Workflow Setup

Set up the source checkout once when you want to invoke Spec Kit skills in Codex while developing this repository. This is local dogfooding state, not release content: the generated Codex skills stay untracked, and the bundled bug workflow is enabled only in this checkout.

First exclude the generated Codex skills from this checkout's local Git status, then initialize Codex skills and add the bug extension through the editable CLI:

```bash
printf '\n# Local Codex dogfooding scaffolding\n.agents/\n' >> .git/info/exclude
mise run pegagio:dogfood init --here --force --integration codex --integration-options="--skills" --script py
mise run pegagio:dogfood extension add bug
```

This requires the `codex` CLI to be installed and available on `PATH`. Do not add `agent-context`: it is only for managing a marked section in an agent instruction file, while this fork's `AGENTS.md` remains manually maintained. Do not add the `git` extension: this fork already has an explicit branch and commit workflow, while that extension creates feature branches and can add commit hooks.

### Optional Codex Workflow Test

The local setup above is sufficient for normal dogfooding. When a change affects `specify init`, generated Codex skills, or a workflow command's first-run behavior, also test it in a disposable project. That verifies the fresh-project experience without relying on existing local scaffolding.

Initialize a temporary project with the Codex skills integration from the editable CLI:

```bash
test_root=$(mktemp -d /tmp/specify-test.XXXXXX)
test_project="$test_root/speckit-test"
mise run pegagio:dogfood init "$test_project" --integration codex --integration-options="--skills" --ignore-agent-tools --script py
```

Open that temporary project in Codex and invoke the generated skills there. The core workflow is installed by `init`; for example, use `$speckit-specify`, then its required follow-on skills in order when the change affects them.

The bug extension is part of the local setup above. Other extensions remain optional test subjects; install one in the disposable project only when the change or manual test exercises its behavior:

```bash
# Test the bundled idea-assessment workflow.
SPECIFY_INIT_DIR="$test_project" mise run pegagio:dogfood extension add assess
```

Install the `git` extension only in a disposable project when changing or testing it. It is not part of the normal Pegagio workflow.

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

The checkout uses two PEP 440 local-version states. Live source dogfooding uses `<base>+pegagio.dev`. A registered build uses `<base>+pegagio.<n>`, with a matching `v<version>` tag. Advance the numeric suffix for every different build; never assign new contents to an existing tag or version.

Do not change the package version for ordinary edits or validation runs. Change it at the two release boundaries: prepare the next numeric suffix only when a clean `main` commit is ready to become a consumable build, then restore `+pegagio.dev` in a follow-up commit after registration. When an upstream sync changes the base version, restore development mode for that base before dogfooding; the first build for the new base is `+pegagio.1`.

Read the current package version with:

```bash
mise run pegagio:get-version
```

The `pegagio:next-version` task derives the next usable numeric build version from `pyproject.toml` and locally available `v<base>+pegagio.<n>` tags. From `+pegagio.dev`, it chooses the next numeric suffix. It keeps an already prepared, untagged numeric suffix instead of skipping it. Preview that decision before writing it:

```bash
mise run pegagio:next-version
```

Use `pegagio:set-version <version>` only when you need to choose a nonstandard version explicitly.

Prepare the next version, retain it in a shell variable for the later checks and tag push, inspect the isolated change, validate it, and commit it:

```bash
mise run pegagio:next-version --write
release_version=$(mise run pegagio:get-version)
printf 'Preparing fork release %s\n' "$release_version"
git diff -- pyproject.toml
mise run pegagio:validate
git add pyproject.toml
git commit -m "chore: prepare fork release"
```

Build and register that exact committed version with one task:

```bash
mise run pegagio:build-register
```

The task refuses a dirty tree, a non-`main` branch, version/tag/artifact/mise collisions, or unexpected CLI version. On success it writes artifacts under `dist/local/<version>/`, records `provenance.json`, registers `pipx:specify-cli@<version>`, and creates a local tag. It removes only artifacts and registrations it created if a later build step fails.

Return the source checkout to explicit live development mode immediately after a successful registration:

```bash
mise run pegagio:resume-development
git diff -- pyproject.toml
git add pyproject.toml
git commit -m "chore: resume Pegagio development"
```

`pegagio:build-register` rejects the `+pegagio.dev` version, so development source cannot be registered accidentally.

Verify the result before using it:

```bash
mise where "pipx:specify-cli@$release_version"
mise exec "pipx:specify-cli@$release_version" -- specify --version
shasum -a 256 "dist/local/$release_version/wheelhouse/specify_cli-$release_version-py3-none-any.whl"
```

Keep the commands in one shell session so `release_version` remains available. If you start a new session, copy the version from the successful `pegagio:build-register` output before running the verification or tag-push commands.

Remote publication remains explicit after this local verification:

```bash
git push origin main:main
git push origin "v$release_version"
```

Pushing the tag shares the source identity, not the locally built wheel or mise registration. Another machine must build and register that exact tag before it can consume the fork release.

## Consume a Fork Release in Other Projects

Every project outside this checkout consumes a registered fork release. It records its selected version in its own `mise.toml`; it never records a Spec Kit checkout path, wheel, branch name, or development link. For example:

```toml
[tools]
"pipx:specify-cli" = "1.0.10.dev0+pegagio.1"
```

From this fork checkout, the `pegagio:consume` task automates release setup in another project. It verifies that the fork build is registered locally, writes the exact project pin, trusts that project's `mise.toml`, installs declared tools, and verifies the selected CLI:

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
mise exec -- specify integration upgrade <key>
mise exec -- specify extension update
```

Specifications, plans, constitutions, source code, and Git history are outside the normal manifest-aware upgrade path. For a Codex skills integration, generated skills are snapshots of the selected version:

```bash
specify init --here --integration codex --integration-options="--skills"
```

Commit generated files only when they are part of the consumer project's intended configuration.

## Additional Development Techniques

Use `uvx` to simulate a user flow from a pushed branch without making it an installed release:

```bash
uvx --from git+https://github.com/pegagio/spec-kit.git@feature-branch specify init demo-branch-test --script py
```

Use the integration scaffold command to create the initial Python package and test skeleton for a new built-in integration:

```bash
mise run pegagio:dogfood integration scaffold my-agent --type markdown
mise run pegagio:dogfood integration scaffold my-agent --type toml
mise run pegagio:dogfood integration scaffold my-agent --type yaml
mise run pegagio:dogfood integration scaffold my-agent --type skills
```

Hyphenated keys become Python-safe package names. The scaffold does not register the integration automatically; review the generated metadata, then add the import and `_register()` call in `src/specify_cli/integrations/__init__.py`.

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
