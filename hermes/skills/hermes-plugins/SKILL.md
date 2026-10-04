---
name: hermes-plugins
version: 1.0.0
description: Use when installing or triaging a third-party Hermes plugin.
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [hermes, plugins, install, security, triage, catalog]
    category: autonomous-ai-agents
    related_skills: [hermes-agent, read-the-damn-docs]
---

# Hermes Plugins

Procedure for extending Hermes with a third-party plugin, and for deciding
whether a security-scan block may be overridden.

## When to Use

- The user asks whether a plugin/skill is installed, enabled, or in the catalog.
- Installing a plugin from a GitHub repo, Git URL, or `owner/repo`.
- `hermes plugins install` returned a security-scan verdict (CAUTION/BLOCKED)
  or asked for `--force`.
- Updating, disabling, or removing an installed plugin.
- `config.yaml` lists a plugin that `hermes plugins list` does not show.

Not for MCP servers (`hermes mcp`), skill authoring, or bundled/plugin
already-working questions — those belong to `hermes-agent`.

## Procedure

1. **Establish what is actually installed before trusting config.**
   `~/.hermes/config.yaml` → `plugins.enabled` can list a plugin that has no
   directory behind it (an orphan entry left by an earlier install). Verify all
   three, in this order:

   ```bash
   hermes plugins list                 # status + version + source (catalog/git/bundled)
   ls ~/.hermes/plugins/               # real directories, one per installed plugin
   cat ~/.hermes/plugins/.install-metadata.json   # provenance: source URL, pinned sha
   ```

   `hermes plugins search <name>` answers a different question — whether the
   name exists in the curated catalog. "No catalog entries matched" means the
   plugin must be installed from a Git URL or `owner/repo`, and also that a
   same-named config entry is likely stale.

2. **Get the install command from the target repo's README**, not from memory.
   Repos ship harness-specific manifests (`.hermes-plugin/plugin.yaml`,
   `.claude-plugin/`, `.codex-plugin/`, …) and a per-harness install section;
   the command differs per harness and changes between releases.

3. **Install:**

   ```bash
   hermes plugins install <owner>/<repo> --enable
   ```

   Catalog sources are pre-reviewed. Git URL / `owner/repo` sources are marked
   `custom (unreviewed)` and are security-scanned at install time.

4. **Triage a scan block before ever reaching for `--force`** (see
   `references/scan-triage.md` for the path-by-path decision table).

5. **Verify the install, then restart sessions.**

   ```bash
   hermes plugins list                 # name present, status enabled
   hermes plugins capabilities         # declared vs granted capabilities
   hermes plugins doctor               # real runtime-contract validation
   ```

   Plugins and their injected bootstrap load at session start — active
   sessions must be restarted, and a long session that has already compacted
   may have lost the injected bootstrap (start a fresh session if skills stop
   triggering).

6. **Maintain:** `hermes plugins update <name>`, `hermes plugins check-updates`,
   `hermes plugins disable <name>` (keep, stop loading),
   `hermes plugins remove <name>` (undo an install — this is the rollback for a
   `--force` override).

## Always-on rules

- **Never hand-edit `config.yaml` to enable, disable, or fix a plugin entry.**
  Use `hermes plugins enable|disable <name>` (or `hermes config set` for other
  keys) — a stray indent in that file corrupts the live gateway.
- **A `BLOCKED — Use --force to override` verdict is a gate, not a failure.**
  Never retry with `--force` silently. Run the triage, report severity counts
  and the payload verdict, and get the user's confirmation first; say plainly
  that `--force` is the override and `hermes plugins remove` is the rollback.
- **Ignore findings in prose when judging risk; weight findings in the
  declared payload.** The scanner is a text matcher over the whole repo,
  including docs, tests, and archived design plans — those matches describe
  files nobody executes, so they cannot justify a block by count alone. A
  finding inside the plugin's own entry code can justify it even when it is
  the only one.
- **Verify a HIGH/CRITICAL finding by reading the raw file before citing it.**
  Fetch the exact line. `rm -rf` in an uninstall doc, a placeholder token in a
  plan document, and `env | grep VAR` in a debugging example are the three
  most common false positives.
- **State findings as counts plus location, not as a verdict word.**
  "22 HIGH" without saying that zero landed in the payload reads as danger;
  "466 findings, 0 in `.hermes-plugin/`" is the fact the user decides on.
