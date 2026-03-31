# hermes-langfuse-tracing

Opt-in Langfuse tracing plugin for Hermes.

This repo is intentionally separate from `NousResearch/hermes-agent`.

Why:
- Hermes core stays clean and upstream-friendly
- the tracing runtime can evolve independently
- Hermes updates do not overwrite your tracing installation
- Hermes can still expose tracing as a skill-driven install flow, similar in spirit to NanoClaw's "skills transform the local install" approach

## What this repo contains

- `langfuse_tracing/__init__.py`
  - the plugin implementation
  - hook handlers for Hermes LLM/tool lifecycle events
  - per-turn trace/span aggregation and payload normalization
- `langfuse_tracing/plugin.yaml`
  - Hermes plugin manifest

## How it fits into Hermes

This repo is the runtime plugin repo.

The Hermes side is expected to expose Langfuse setup through the normal Hermes setup/install surface rather than making users manually copy plugin files.

Concretely, the Hermes flow should be:
1. the user goes through Hermes setup or installs the official `langfuse-tracing` optional skill
2. Hermes asks for or detects the Langfuse settings it needs
3. Hermes installs the `langfuse` Python package if needed
4. Hermes fetches this repo
5. Hermes copies `langfuse_tracing/__init__.py` and `langfuse_tracing/plugin.yaml` into:
   - `$HERMES_HOME/plugins/langfuse_tracing/`
6. Hermes writes Langfuse env vars into `$HERMES_HOME/.env`
7. Hermes verifies `hermes plugins list`
8. on the next Hermes start, the plugin is discovered and becomes active

So the user experience should feel like:
- use Hermes setup / Hermes skill install
- Hermes changes the local install/profile for you
- Hermes loads this repo's plugin code at startup

That is the key NanoClaw-like idea here: the integration is presented to the user through the assistant's setup/skill system, while the runtime implementation still lives in a separate repo.

## How this relates to `hermes setup`

This repo is not the setup wizard itself.

Instead:
- Hermes owns the setup UX
- this repo owns the runtime plugin code
- Hermes setup should point at this repo as the source of truth for the plugin files

Think of the boundary like this:

### Hermes setup is responsible for
- asking whether the user wants Langfuse tracing
- collecting or validating Langfuse credentials/base URL
- installing the Python dependency if needed
- fetching/copying the plugin into the active profile
- writing env vars into the correct `$HERMES_HOME/.env`
- telling the user to restart Hermes if needed

### This repo is responsible for
- the actual Hermes plugin manifest
- the hook handlers and trace/span logic
- payload normalization and fail-open behavior
- plugin-side compatibility with Hermes hook evolution

That split is important because it keeps Hermes setup clean and user-facing, while keeping the tracing runtime independently versioned.

## Architecture

Open `docs/hermes-langfuse-flow.excalidraw` in Excalidraw to edit the diagram.

High-level flow:
- Hermes optional skill installs this plugin into the active profile
- Hermes startup discovers the plugin in `$HERMES_HOME/plugins/`
- Hermes invokes plugin hooks around LLM calls and tool calls
- the plugin emits traces/spans to Langfuse when enabled
- if env vars or dependency are missing, the plugin fails open and stays dormant

## Excalidraw diagram

A rendered summary of the Excalidraw file:

1. `hermes skills install official/observability/langfuse-tracing`
2. skill installer script fetches this repo
3. plugin files land in `$HERMES_HOME/plugins/langfuse_tracing/`
4. Hermes starts and discovers the plugin
5. Hermes calls `pre_llm_call`, `post_llm_call`, `pre_tool_call`, `post_tool_call`
6. plugin groups activity into one Hermes turn trace with nested observations
7. Langfuse receives the trace data

## Installation model

### Recommended: install via Hermes skill

Use the Hermes optional skill so the install is profile-aware and repeatable.

Expected result:
- plugin files copied into the active Hermes profile
- Langfuse env vars written to `$HERMES_HOME/.env`
- verification via `hermes plugins list`

### Manual install

If you want to install directly from this repo:

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

## Required environment variables

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

Fallback aliases are supported in the plugin code:
- `CC_LANGFUSE_*`
- bare `LANGFUSE_*`

Resolution priority is:
- `HERMES_*`
- `CC_*`
- bare `LANGFUSE_*`

## What gets traced

Per Hermes turn, the plugin can capture:
- root Hermes turn trace
- LLM generations
- tool calls
- normalized tool outputs
- session/task metadata
- optional environment/release tagging

The plugin also normalizes some payloads so traces stay useful:
- parses JSON tool payloads when possible
- preserves trailing hint text
- summarizes `read_file` output as structured previews
- omits large raw binary/base64 payloads

## Fail-open behavior

This plugin is designed to fail open.

That means:
- if `langfuse` is not installed, Hermes still runs
- if env vars are missing, Hermes still runs
- if Langfuse is unreachable, Hermes still runs
- if plugin initialization fails, Hermes still runs

Tracing simply becomes dormant instead of breaking the agent.

## How Hermes updates work with this plugin installed

This is the most important design point.

### Hermes updates

When you run:

```bash
hermes update
```

Hermes updates the Hermes codebase, not this plugin repo and not the installed plugin directory under `$HERMES_HOME/plugins/`.

That means upstream Hermes pulls should not clobber your Langfuse plugin installation.

Conceptually:
- Hermes repo updates core code
- installed plugin lives under Hermes home
- external plugin source lives in this repo
- those are separate layers

This is the same core operational idea NanoClaw uses for skill-driven local customization:
- keep the customization path out of the mainline core surface as much as possible
- let install/update actions be explicit and local

### Plugin updates

Plugin updates are separate from Hermes updates.

You have two good workflows.

#### Workflow A: Hermes skill-managed copy install

1. update this repo
2. rerun the Hermes `langfuse-tracing` installer skill/script
3. it recopies the latest plugin files into `$HERMES_HOME/plugins/langfuse_tracing/`
4. restart Hermes

Pros:
- simple
- profile-aware
- matches the intended Hermes skill UX

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

Recommended expectation:
- Hermes core can update independently
- most Hermes updates should keep the plugin working
- if hook contracts evolve, update this repo and reinstall/restart

## Typical update playbook

### Safe normal case

```bash
hermes update
# Hermes core updates
# plugin install remains untouched
# restart Hermes and keep working
```

### If you also want the latest plugin changes

Copy-install workflow:

```bash
cd /path/to/hermes-langfuse-tracing
git pull
# rerun Hermes langfuse-tracing installer
# restart Hermes
```

Symlink workflow:

```bash
cd /path/to/hermes-langfuse-tracing
git pull
# restart Hermes
```

## Verification

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

## Development notes

This repo is meant to be the canonical source for the plugin runtime.

Hermes should only contain:
- the optional skill
- installer/update helper logic
- tests for the skill installer path

That separation keeps upstream Hermes clean while still making the integration easy to install.

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
- add more explicit background-review / turn-type tagging once the corresponding Hermes hooks are standardized
