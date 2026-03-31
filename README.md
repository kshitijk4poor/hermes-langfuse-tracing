# hermes-langfuse-tracing

Opt-in Langfuse tracing plugin for Hermes.

This repository is the canonical source for the Langfuse tracing runtime plugin. In the intended Hermes architecture, Hermes setup or an optional skill installs this plugin into the active profile; Hermes core and this plugin are updated independently.

## TL;DR

- Recommended install path: use Hermes setup or the official `langfuse-tracing` optional skill when that installer path is available in your Hermes build.
- Fallback install path: manually copy this repo's plugin files into `$HERMES_HOME/plugins/langfuse_tracing/`.
- Configuration lives in `$HERMES_HOME/.env`.
- `hermes update` updates Hermes core only. It does not overwrite the installed plugin under `$HERMES_HOME/plugins/`.
- Plugin updates are separate: either rerun the installer or use a symlink workflow and `git pull` this repo.

## Installation

### Recommended: Hermes setup or optional skill

This repo is meant to be installed through the normal Hermes setup/install surface rather than by making users manually copy plugin files.

The intended Hermes flow is:
1. The user opts into Langfuse tracing in `hermes setup` or installs the official `langfuse-tracing` optional skill.
2. Hermes asks for or detects the Langfuse settings it needs.
3. Hermes installs the `langfuse` Python package if needed.
4. Hermes fetches this repo.
5. Hermes copies `langfuse_tracing/__init__.py` and `langfuse_tracing/plugin.yaml` into:
   - `$HERMES_HOME/plugins/langfuse_tracing/`
6. Hermes writes Langfuse env vars into `$HERMES_HOME/.env`.
7. Hermes verifies `hermes plugins list`.
8. On the next Hermes start, the plugin is discovered and becomes active.

If your current Hermes build does not expose that installer path yet, use the fallback manual install below.

### Fallback: manual install

```bash
PLUGIN_DIR="$HOME/.hermes/plugins/langfuse_tracing"
mkdir -p "$PLUGIN_DIR"
cp langfuse_tracing/__init__.py "$PLUGIN_DIR/__init__.py"
cp langfuse_tracing/plugin.yaml "$PLUGIN_DIR/plugin.yaml"
```

For named profiles, install into:

```bash
$HOME/.hermes/profiles/<profile>/plugins/langfuse_tracing/
```

Then set env vars in the matching profile's `$HERMES_HOME/.env`.

## Configuration

Tracing stays dormant unless enabled and both keys are present.

Required:

```bash
HERMES_LANGFUSE_ENABLED=true
HERMES_LANGFUSE_PUBLIC_KEY=pk-lf-...
HERMES_LANGFUSE_SECRET_KEY=sk-lf-...
```

Optional:

```bash
HERMES_LANGFUSE_BASE_URL=https://cloud.langfuse.com
HERMES_LANGFUSE_ENV=development
HERMES_LANGFUSE_RELEASE=v0.1.0
HERMES_LANGFUSE_SAMPLE_RATE=1.0
HERMES_LANGFUSE_MAX_CHARS=12000
HERMES_LANGFUSE_DEBUG=true
```

Compatibility aliases supported by the plugin:
- `CC_LANGFUSE_*`
- bare `LANGFUSE_*`
- `TRACE_TO_LANGFUSE` as a legacy enable flag alias

Resolution priority is:
- `HERMES_*`
- `CC_*`
- bare `LANGFUSE_*`

## Verify installation

After install or update:
1. restart Hermes
2. run:

```bash
hermes plugins list
```

3. verify `langfuse_tracing` appears
4. run a simple Hermes prompt
5. confirm Langfuse shows:
- one Hermes turn trace
- nested LLM spans
- nested tool spans when tools are used

If debugging is needed:

```bash
HERMES_LANGFUSE_DEBUG=true hermes chat -q "hello"
```

## How it fits into Hermes

This repo is the runtime plugin repo, not the setup wizard.

There are three layers:
- Hermes setup / optional skill
  - installer and user-facing configuration layer
- this repo
  - canonical source for the runtime plugin code
- `$HERMES_HOME/plugins/langfuse_tracing/`
  - installed local plugin artifact that Hermes discovers at startup

### Hermes setup is responsible for
- asking whether the user wants Langfuse tracing
- collecting or validating Langfuse credentials and base URL
- installing the Python dependency if needed
- fetching and copying the plugin into the active profile
- writing env vars into the correct `$HERMES_HOME/.env`
- telling the user to restart Hermes if needed

### This repo is responsible for
- the Hermes plugin manifest
- the hook handlers and trace/span logic
- payload normalization and fail-open behavior
- plugin-side compatibility with Hermes hook evolution

That split is the NanoClaw-like packaging idea here: the integration is presented through the assistant's setup/skill system, while the runtime implementation still lives in a separate repo.

## Updates

### Hermes core updates

When you run:

```bash
hermes update
```

Hermes updates the Hermes codebase, not this repo and not the installed plugin directory under `$HERMES_HOME/plugins/`.

That means Hermes core updates should not clobber your Langfuse plugin installation.

Conceptually:
- Hermes repo updates core code
- installed plugin lives under Hermes home
- external plugin source lives in this repo
- those are separate layers

### Plugin updates

Plugin updates are separate from Hermes updates.

#### Workflow A: installer-managed copy install

1. update this repo source
2. rerun the Hermes `langfuse-tracing` installer path
3. it recopies the latest plugin files into `$HERMES_HOME/plugins/langfuse_tracing/`
4. restart Hermes

Pros:
- simple
- profile-aware
- matches the intended Hermes setup/skill UX

#### Workflow B: symlink for active development

Instead of copying files into the plugin directory, symlink this repo:

```bash
PLUGIN_DIR="$HOME/.hermes/plugins/langfuse_tracing"
rm -rf "$PLUGIN_DIR"
ln -s /path/to/hermes-langfuse-tracing/langfuse_tracing "$PLUGIN_DIR"
```

Then updates are just:

```bash
cd /path/to/hermes-langfuse-tracing
git pull
```

and restart Hermes.

Pros:
- best for development
- no recopy step

### Compatibility expectations

The plugin uses Hermes hook signatures and intentionally tries to absorb upstream evolution safely.

Still, when Hermes core adds or changes hook kwargs, you may need a plugin update from this repo.

Practical expectation:
- Hermes core can update independently
- most Hermes updates should keep the plugin working
- if hook contracts evolve, update this repo and reinstall or restart

## Tracing behavior

Per Hermes turn, the plugin can capture:
- root Hermes turn trace
- LLM generations
- tool calls
- normalized tool outputs
- session and task metadata
- optional environment and release tagging

The plugin also normalizes some payloads so traces stay useful:
- parses JSON tool payloads when possible
- preserves trailing hint text
- summarizes `read_file` output as structured previews
- omits large raw binary or base64 payloads

## Fail-open behavior

This plugin is designed to fail open.

That means:
- if `langfuse` is not installed, Hermes still runs
- if env vars are missing, Hermes still runs
- if Langfuse is unreachable, Hermes still runs
- if plugin initialization fails, Hermes still runs

Tracing simply becomes dormant instead of breaking the agent.

## Architecture diagram

Open `docs/hermes-langfuse-flow.excalidraw` in Excalidraw to edit the diagram.

High-level flow:
1. Hermes setup or the `langfuse-tracing` optional skill installs this plugin into the active profile.
2. Hermes startup discovers the plugin in `$HERMES_HOME/plugins/`.
3. Hermes invokes plugin hooks around LLM calls and tool calls.
4. The plugin emits traces and spans to Langfuse when enabled.
5. If env vars or dependency are missing, the plugin stays dormant.

## Repository layout

```text
hermes-langfuse-tracing/
├── README.md
├── docs/
│   └── hermes-langfuse-flow.excalidraw
└── langfuse_tracing/
    ├── __init__.py
    └── plugin.yaml
```

## Future improvements

Potential next steps:
- publish tagged releases for plugin compatibility with Hermes versions
- add a small self-check CLI for plugin diagnostics
- document a version matrix: Hermes version <-> plugin version
- add more explicit background-review and turn-type tagging once the corresponding Hermes hooks are standardized
