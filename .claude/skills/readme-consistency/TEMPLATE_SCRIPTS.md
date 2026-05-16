# {ParentProject}/scripts

{1-2 sentence narrative paragraph: what this scripts directory holds, who calls them, and how they fit into the parent project. The opening paragraph serves as the implicit overview — no explicit `## Overview` header is required for sub-component READMEs.}

## Files

| Script | Purpose |
|---|---|
| `script-name.sh` | {what it does, who invokes it, any flags worth knowing} |
| `script-name.py` | {what it does} |

(Or use `## Project Structure` / `## Layout` if the directory has a richer tree
than a flat file list.)

## Usage

```bash
# Most-common invocation, with concrete arguments and the directory it should
# be run from.
bash {ParentProject}/scripts/script-name.sh --flag value
```

(Synonyms accepted by the audit: `## When to run`, `## When to use`,
`## Development`, `## Common edits`, `## Typical workflow`, `## How to run`,
`## How to use`. Pick whichever reads most naturally.)

## See Also

- [{ParentProject}](../) — parent project README
- {sibling scripts dir if applicable}

---

## Audit notes (not part of the template)

`audit.sh` applies a 3-section template to `*/scripts` directories:

- **Overview** — explicit `## Overview` header, OR a narrative paragraph
  between the `# Title` and the first `## Section` header.
- **Project Structure** — `## Project Structure`, `## Files`, `## Layout`,
  or `## Current contents`.
- **Usage** — see the synonym list above.

Architecture / Features / Configuration / API Endpoints / Ports / Deployment /
See Also are **not** required for scripts dirs — they don't apply.
