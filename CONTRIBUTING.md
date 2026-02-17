# Contributing

Thanks for contributing to the grimoire registry! Every grimoire you add saves other developers and AI agents from redundant codebase exploration.

## Quick start

1. Generate a grimoire for a public GitHub repo:

```bash
grimoire analyze <name> --github owner/repo | claude
# or
grimoire analyze <name> --github owner/repo --mode api
```

2. Fork this repo

3. Copy your grimoire into the correct path:

```bash
mkdir -p <owner>/<repo>
cp ~/.grimoire/projects/<name>/grimoire.json <owner>/<repo>/
cp -r ~/.grimoire/projects/<name>/topics/ <owner>/<repo>/topics/
```

4. Open a PR

## Requirements

- The project must be a **public GitHub repo**
- `grimoire.json` must include `github` (owner/repo format) and `sourceType: "github"`
- Must include at least 5 topics in the `topics/` directory
- Topics must have valid YAML frontmatter (title, slug, description, order, category)
- No API keys, secrets, or private information in any file

## Directory naming

Use the GitHub owner/repo as the directory path:

| GitHub repo | Registry path |
|-------------|---------------|
| `tim-smart/effect-atom` | `tim-smart/effect-atom/` |
| `effect-ts/effect` (packages/sql) | `effect-ts/effect/sql/` |
| `colinhacks/zod` | `colinhacks/zod/` |

## Topic quality

Good grimoire topics:

- Focus on what AI agents need to write correct code
- Use real code examples from the codebase, not generic ones
- Cover architecture decisions — why things are structured the way they are
- Include file pointers — which files to read for specific concerns
- Range from general (overview, setup) to specific (modules, patterns)

## Updating existing grimoires

To update an existing grimoire, regenerate it against the latest version of the source repo and open a PR with the changes.
