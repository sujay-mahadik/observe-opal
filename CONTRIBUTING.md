# Contributing to observe-opal

This plugin ships three skills. Each skill lives in its own subdirectory under `skills/` and is a self-contained directory of Markdown files — no code, no build step.

```
observe-opal/
├── .claude-plugin/
│   └── plugin.json          ← plugin manifest (name, version, keywords)
├── skills/
│   ├── opal-query/
│   │   ├── SKILL.md         ← skill prompt + instructions
│   │   └── references/
│   │       ├── syntax.md    ← complete OPAL verb/function reference
│   │       └── datasets.md  ← Observe dataset schemas
│   ├── elastic-event-to-metric/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── elastic-endpoint.md
│   └── grok-to-opal/
│       ├── SKILL.md
│       └── references/
│           └── grok-patterns.md
├── README.md
└── CONTRIBUTING.md
```

---

## How skills work

A skill is invoked when Claude matches the user's message against the `description` field in the SKILL.md frontmatter. When triggered, Claude reads the full SKILL.md (and any referenced files) before responding.

**SKILL.md structure:**
```markdown
---
name: skill-name
description: >
  Trigger phrases and description used for skill matching.
  Be specific — vague descriptions cause missed or wrong triggers.
---

# Skill Title

Instructions for Claude...
```

References files (in `references/`) are read by Claude when the SKILL.md explicitly tells it to (`Read references/syntax.md for the complete syntax guide`). They are not auto-loaded — they must be referenced by name in SKILL.md.

---

## Making changes

### Update an existing skill

1. Edit `skills/<skill-name>/SKILL.md` — the main instruction prompt
2. Edit files under `skills/<skill-name>/references/` — supporting reference tables and examples
3. Bump the version in `.claude-plugin/plugin.json` (semver: patch for fixes, minor for new capabilities, major for breaking changes)
4. Test locally (see below)
5. Open a PR — include a before/after example showing the improvement

### Add a new skill

1. Create `skills/<your-skill>/SKILL.md` with frontmatter `name` and `description`
2. Add any reference files under `skills/<your-skill>/references/`
3. Bump the plugin version in `.claude-plugin/plugin.json`
4. Add a section to `README.md` documenting the new skill and its trigger phrases
5. Test locally, then open a PR

### Add new OPAL verbs or functions

The OPAL syntax reference lives in `skills/opal-query/references/syntax.md`. To add a newly documented verb or function:

1. Find the right section in `syntax.md` (verbs are grouped by category)
2. Add the entry following the existing pattern — include signature, description, and at least one example
3. If the verb/function is relevant to field extraction or metrics, also update `skills/elastic-event-to-metric/SKILL.md` or `skills/opal-query/SKILL.md` as appropriate

### Add new Grok pattern mappings

Pattern mappings live in `skills/grok-to-opal/references/grok-patterns.md`.

- **New primitive pattern**: add a row to the relevant table (Generic matchers, Numbers, Networking, etc.) with the Grok name, OPAL regex equivalent, and a notes column if there are caveats
- **New composite log format**: add a pre-built OPAL translation under "Pre-built OPAL translations" — include the Grok pattern name, field list comment, and the full `extract_regex` statement
- **Regex constraint reminder**: OPAL does not support `(?:...)` non-capturing groups. All patterns in this file must use only named groups `(?P<name>...)`, character classes `[^x]+`, or plain quantifiers. See the constraints section in `skills/opal-query/SKILL.md` for the full list.

---

## Testing locally

### Install the plugin from your local clone

```shell
# In Claude Code:
/plugin marketplace add ./path/to/observe-opal

# Or install the packaged .plugin file directly:
/plugin install ./observe-opal.plugin
```

### Validate before committing

```bash
# From the observe-opal directory:
claude plugin validate .
```

This checks `plugin.json`, skill frontmatter, and any `hooks.json` for schema errors. Fix all errors before opening a PR. Warnings (non-blocking) are still worth addressing.

### Manual smoke tests

After installing locally, test each changed skill with a representative prompt:

| Skill | Test prompt |
|---|---|
| `opal-query` | "Write an OPAL query to count 5xx errors per service" |
| `opal-query` | "Extract fields from: `10.0.0.1 - - [28/Mar/2026:10:22:15 +0000] \"GET / HTTP/1.1\" 200 1234`" |
| `elastic-event-to-metric` | Paste a raw Logstash/Metricbeat JSON event and ask to convert it to a metric interface |
| `grok-to-opal` | "Convert this grok: `%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}`" |

---

## Version policy

The version in `.claude-plugin/plugin.json` controls update detection — users only receive an update if the version changes.

| Change type | Version bump |
|---|---|
| Fix wrong regex, typo, broken example | Patch (`0.2.0` → `0.2.1`) |
| New pattern, new skill mode, new trigger phrases | Minor (`0.2.0` → `0.3.0`) |
| New skill, restructured prompt, breaking change to output format | Major (`0.2.0` → `1.0.0`) |

---

## Key constraints to remember

- **No `(?:...)` non-capturing groups** in any regex — OPAL's engine rejects them with `no argument for repetition operator: ?`. Use named groups `(?P<name>...)`, negated character classes `[^x]+`, or restructure the pattern.
- **Timestamps always use the two-step pattern**: extract as a raw string first, then `parse_isotime()` in a follow-up `make_col`. Never embed optional timezone/fractional-second groups in the main regex.
- **`extract_regex` is a verb, not a function** — `extract_regex body, /(?P<field>pattern)/`, not `make_col field: extract_regex(...)`.
- **`interface "metric"` requires exactly two columns**: `metric` (string) and `value` (float64). Use `flatten_leaves` + rename/cast to get there.
- **`set_timestamp` for event-shaped data** (not `set_valid_from`, which is deprecated for events).
- **`parse_isotime()` for ISO string timestamps** (not `timestamp_s()`, which takes int64 epoch seconds).
