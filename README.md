# grimoire registry

Pre-built grimoires for popular open-source projects. Used by the [grimoire CLI](https://github.com/llm-grimoire/cli) to give AI agents instant codebase knowledge.

## Usage

```bash
npm install -g @llm-grimoire/cli
grimoire add tim-smart/effect-atom
grimoire show effect-atom overview
```

## Structure

Grimoires are namespaced by GitHub `owner/repo`. Monorepo sub-packages get nested directories:

```
packages/
  tim-smart/
    effect-atom/
      grimoire.json
      topics/
        00-overview.md
        01-architecture.md
        ...
  effect-ts/
    effect/
      sql/
        grimoire.json
        topics/...
      ai/
        grimoire.json
        topics/...
```

## grimoire.json

Each grimoire must include a `grimoire.json`:

```json
{
  "name": "effect-atom",
  "description": "Atomic state management for Effect",
  "version": "0.1.0",
  "github": "tim-smart/effect-atom",
  "sourceType": "github",
  "topicsDir": "topics"
}
```

For monorepo sub-packages, add `path`:

```json
{
  "name": "effect-sql",
  "description": "SQL toolkit for Effect",
  "version": "0.1.0",
  "github": "effect-ts/effect",
  "path": "packages/sql",
  "sourceType": "github",
  "topicsDir": "topics"
}
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Content in this registry is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/).
