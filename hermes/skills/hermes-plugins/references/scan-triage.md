# Security-scan triage for `hermes plugins install`

When install stops with `Decision: BLOCKED — … Use --force to override`, work
through these steps in order. Do not skip to `--force` and do not skip the
report — the user is the one who decides.

## 1. Capture the full scan

```bash
hermes plugins install <owner>/<repo> --enable > /tmp/<name>-scan.txt 2>&1
```

The refusal exits non-zero and cleans up its clone; re-running is safe and
read-only.

## 2. Group findings by path

```python
import re, collections
pat = re.compile(r'^\s*(CRITICAL|HIGH|MEDIUM|LOW)\s+([a-z_]+)\s+(\S+?):\d+', re.M)
findings = pat.findall(open('/tmp/<name>-scan.txt').read())
print(collections.Counter(f[0] for f in findings))            # severity totals
print(collections.Counter(f[2].split('/')[0] for f in findings))  # top-level dirs
```

## 3. Split payload from prose

| Path | Weight |
|---|---|
| `.hermes-plugin/` (`plugin.yaml`, `__init__.py`), `hooks/`, `index.js`, `scripts/` | **Runtime payload** — read it |
| `skills/**/SKILL.md`, `references/` | Loaded as model context, not executed — check for instruction-level issues only |
| `docs/`, `tests/`, `*.md`, `plans/`, `specs/`, `README*` | Prose and fixtures — text matches, effectively noise |
| `.claude-plugin/`, `.codex-plugin/`, `.opencode/`, `.pi/` | Other harnesses — not executed by Hermes |

Fetch the declared payload directly when the clone is gone:

```bash
curl -fsSL https://raw.githubusercontent.com/<owner>/<repo>/main/.hermes-plugin/plugin.yaml
```

`plugin.yaml` tells you what actually runs: `provides_hooks` lists the hook
points, and the sibling Python module is the code behind them.

## 4. Verify every HIGH/CRITICAL before reporting it

Read the cited line. Common false positives:

- `rm -rf <path>` inside an uninstall/directions document
- placeholder tokens (`abababab…`) in a design doc discussing token handling
- `env | grep VAR` inside a debugging example
- `npm install …` / `git clone …` quoted as an install instruction
- `../../` path joins in example code (`traversal` on a doc snippet)

If the cited file does not exist in the payload, say so explicitly.

## 5. Report, then ask

Report: severity counts, files-by-top-directory, whether the declared payload
has zero findings, and the fact that `--force` overrides the gate while
`hermes plugins remove <name>` rolls it back. Then ask for confirmation.

## When NOT to recommend `--force`

Any finding inside the declared payload — `.hermes-plugin/`, hooks, the entry
module, `plugin.yaml` — deserves a line-by-line read first, regardless of its
severity label. That is the code that runs on every session start.
