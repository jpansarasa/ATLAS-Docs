# {ParentProject}/config

{1-2 sentence narrative paragraph: what this directory holds, how it's mounted
into the service, how environment overrides interact with it, and what stays out
of it. The opening paragraph serves as the implicit overview — no explicit
`## Overview` header is required for sub-component READMEs.}

## Files

| File | Purpose |
|---|---|
| `app.yaml` | {what it controls, header pointers for full key list} |

(Or use `## Project Structure` / `## Layout` / `## Current contents` if a tree
fits better — e.g. `instruments/<symbol>.json` per-file fan-out.)

## Common edits

| Goal | Key(s) |
|---|---|
| {scenario} | `key.path` |

(Optional — only include if there's a non-trivial set of operator-facing knobs.
Otherwise the file headers and per-key comments are the canonical reference.)

## See Also

- [{ParentProject}](../) — parent project README, full Configuration section
- {related dir}

---

## Audit notes (not part of the template)

`audit.sh` applies a 2-section template to `*/config` directories:

- **Overview** — explicit `## Overview` header, OR a narrative paragraph
  between the `# Title` and the first `## Section` header.
- **Project Structure** — `## Project Structure`, `## Files`, `## Layout`,
  or `## Current contents`.

Everything else (Architecture / Features / Configuration / API Endpoints /
Ports / Deployment / Development / See Also) is **not** required — the parent
project's full README owns those, and the config dir's content speaks for
itself through the file headers.
