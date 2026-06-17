# Godot Assistant Pro — AI Coding Assistant & Game Forge Agent

An in-editor AI plugin for Godot 4.6 that pairs a manual coding assistant with an autonomous **Game Forge Agent** — describe a game in plain English and the agent plans, generates, builds, and wires up the scenes and scripts for you. Backed by a pluggable LLM relay (DeepSeek / Qwen) and a growing library of production-ready code, scene references, and free 3D assets.

## What's in here

- **AI Coding Assistant** — a manual, tab-based tool for one-off GDScript generation, scene (`.tscn`) generation, and free-form chat with the AI.
- **Game Forge Agent** — a multi-phase autonomous pipeline: plan → generate scripts → generate scene builders → run builders → assemble final scenes, with retry handling and session persistence.
- **Resource Library** — a catalog of vetted scripts, scene templates, and free GLB models the agent can pull in instead of generating everything from scratch.

## How it works

The plugin never talks to an LLM provider directly. Every prompt goes through a **relay server** (your own backend, configured per-project) which forwards it to DeepSeek or Qwen and returns the raw text response. This keeps API keys out of the Godot project entirely.

```
Editor UI (agent_panel.gd / plugin_m.gd)
        │
        ▼
relay_panel.gd  ──POST /execute-js-direct──▶  Relay Server  ──▶  DeepSeek / Qwen
        │                                                              │
        ◀──────────────────────── plain text reply ────────────────────┘
        ▼
agent_executor.gd  → sanitizes output → writes .gd / .tscn → rescans filesystem
```

### Game Forge Agent pipeline

1. **Plan** — the user's prompt is sent against a 2D or 3D planning system prompt (auto-detected from the prompt, or set manually) and returns a JSON manifest of files to create: logic scripts, builder scripts, and their output scenes.
2. **Generate** — each file in the manifest is generated one at a time against a strict, Godot-4.6-specific system prompt (correct node types, no redeclaring built-ins like `velocity`, four-direction movement rules, typed variables, etc.).
3. **Build** — builder scripts are `@tool extends EditorScript` files that assemble a scene tree in code and save it with `ResourceSaver.save()`. The agent runs each builder, polls for its output `.tscn`, and times out / retries if it doesn't appear.
4. **Assemble** — once all files are ready, the manifest is marked complete and the project is left with clean, owner-correct scenes and scripts.

Sessions are checkpointed to `manifest.json`, so a build can be closed and resumed without restarting the plan. If the agent can't proceed confidently, it pauses and asks the user a clarifying question instead of guessing.

## Requirements

- Godot 4.6
- A deployed relay server reachable over HTTPS, exposing `/status` (health check) and `/execute-js-direct` (prompt forwarding) — see [Configuration](#configuration)
- API access to at least one supported provider (DeepSeek and/or Qwen) configured on the relay side
- (Optional, for the resource library) a Postgres database — e.g. [Neon](https://neon.tech) — for the script/scene catalog, and a public file host for 3D assets (see [Resource & Asset Library](#resource--asset-library))

## Installation

1. Copy the `addons/godotassistantpro/` folder into your project's `res://addons/` directory.
2. In Godot, go to **Project → Project Settings → Plugins** and enable **Godot Assistant Pro**.
3. Create `res://addons/godotassistantpro/config.cfg` (see below) with your relay URL and token.
4. Reload the project. Two new entries appear under **Project → Tools**: *Godot Assistant* and *Game Forge Agent*. A new **AI Assistant** panel appears in the bottom dock.

## Configuration

`config.cfg`:

```ini
[relay]
relay_url = https://your-relay-server.example.com
relay_token = your-api-key-here
```

- `relay_url` accepts `http(s)://` or `ws(s)://` — the plugin normalizes WebSocket-style URLs to HTTPS internally since requests are plain HTTP.
- The bottom **AI Assistant** panel shows live connection status and lets you switch between **DeepSeek** (fast, coding-focused) and **Qwen** (slower, longer context) at any time; both windows share whichever provider is currently selected.

## Usage

### Godot Assistant (manual mode)

Open via **Project → Tools → Godot Assistant**. Three tabs:

| Tab | Purpose |
|---|---|
| GDScript | Describe a script, get a complete `.gd` file back. Open/save/copy buttons included. |
| SceneGenerator | Describe a scene, get a complete `.tscn` file back. |
| Chat | Free-form Q&A about GDScript, scene structure, or Godot features. |

A checkbox toggles between the **full system prompt** (detailed Godot 4.6 rules — recommended for unfamiliar requests) and a **short prompt** (faster, assumes the model already knows the rules — good for quick iterations).

### Game Forge Agent (autonomous mode)

Open via **Project → Tools → Game Forge Agent**.

1. Pick **2D Game** or **3D Game** (or just mention it in your prompt — the agent detects it).
2. Describe the game, e.g. *"a 2D platformer where the player collects coins and avoids spikes."*
3. Click **Generate Plan**. The agent proposes a file manifest, shown as a checklist.
4. If the request is ambiguous, the agent pauses with **ASK_USER** and a reply box — answer it to continue planning.
5. The agent works through the checklist automatically: scripts first, then builders, then runs each builder and waits for its scene output.
6. Watch progress in the **Activity Log** and the completion **progress bar**. Use **Stop** to halt, or **New Session** to discard the current manifest and start over.

Generated scenes follow a fixed structural convention (e.g. 2D scenes always use a `Node2D` root with `StaticBody2D` ground, `CharacterBody2D` player, and `Camera2D` siblings; 3D defaults to a separate reusable `player.tscn` instanced into `main.tscn`) so output stays predictable and easy to hand-edit afterward.

## Resource & Asset Library

Rather than generating every file from a blank prompt, the agent can draw on a curated library of known-good resources:

- **Scripts & scene templates** — full reference `.gd` and `.tscn` source stored as rows in a Postgres catalog (Neon). The planning/generation step can query this catalog by keyword and either reuse a snippet directly or use it as a grounded example, instead of relying purely on the model's own generation.
- **Free 3D models (`.glb`)** — small-to-mid-sized models (KB up to a few MB) hosted in a public GitHub repo and served through the [jsdelivr](https://www.jsdelivr.com/) GitHub CDN for reliable, cached delivery:

  ```
  https://cdn.jsdelivr.net/gh/<user>/<repo>@main/models/<file>.glb
  ```

  A flat `manifest.json` in the same repo maps tags/descriptions to file paths so the agent can match a request like *"forest with trees"* to the right model without guessing filenames:

  ```json
  {
    "models": [
      { "name": "pine_tree", "tags": ["tree", "forest", "nature"], "path": "models/tree_pine.glb", "size_kb": 420 },
      { "name": "wooden_crate", "tags": ["prop", "crate", "box"], "path": "models/crate.glb", "size_kb": 180 }
    ]
  }
  ```

  When the agent selects a model, it downloads the binary via `HTTPRequest` (using `store_buffer`, never a string write — GLB data isn't valid UTF-8) into `res://assets/models/`, rescans the filesystem, and references the resulting `PackedScene` as an `ext_resource` in the generated `.tscn`.

> Asset license note: confirm and document the license/attribution for every model in the repo (e.g. CC0, CC-BY) before shipping a game that bundles them — add a per-asset `license` field to the manifest if any are not CC0.

## Project structure

```
addons/godotassistantpro/
├── plugin.cfg
├── godotassistant.gd      # EditorPlugin entry point — registers windows, menu items, links panels
├── plugin_m.gd            # Manual Assistant window (GDScript / Scene / Chat tabs)
├── plugin_m.tscn          # UI scene for the manual Assistant window
├── agent_panel.gd         # Game Forge Agent window — planning, generation, build pipeline, UI
├── agent_executor.gd      # File writing, manifest persistence, TSCN/GDScript sanitization
├── relay_panel.gd         # Bottom dock panel — relay connection, provider selection, send_prompt()
├── config.cfg             # relay_url / relay_token (not committed — add to .gitignore)
├── manifest.json          # Active Game Forge Agent session state (auto-generated)
└── builders/              # Temporary @tool EditorScript builder files (auto-generated, run, cleaned up)
```

## Known limitations

- The relay's `/status` and `/execute-js-direct` endpoints must be implemented and reachable; the plugin does not include a backend.
- Builder execution relies on polling for an output file within a fixed timeout (`BUILDER_TIMEOUT`, default 15s) — very slow model responses on complex scenes may need this raised.
- Only two providers are wired up today (DeepSeek, Qwen); adding another means extending `PROVIDER_COMMANDS` in `relay_panel.gd`.
- The resource catalog (Neon) and the GLB CDN catalog are currently separate lookups — there's no unified search across code, scenes, and 3D assets yet.

## Roadmap

- Unified catalog search across scripts, scenes, and models in a single query
- Versioned/pinned asset URLs (commit SHA instead of branch) to avoid CDN cache staleness on updates
- Additional provider support in the relay layer
- Inline asset preview before the agent commits to using a model

## License

Add your license here. If the asset library bundles third-party 3D models, list their individual licenses/attributions separately from the plugin's own license.
