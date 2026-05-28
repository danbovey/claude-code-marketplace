# `danbovey` — Claude Code plugin marketplace

A personal marketplace of [Claude Code](https://www.anthropic.com/claude-code) plugins.

## Install the marketplace

Inside Claude Code:

```
/plugin marketplace add danbovey/claude-code-marketplace
```

Then install any plugin from it as `<plugin>@danbovey`. Browse plugins below.

## Plugins

| Plugin | Description | Install |
|---|---|---|
| [`artifacts`](./plugins/artifacts) | Save shareable Claude Code outputs (docs, PR descriptions, review comments, scripts, configs) to a private Notion database with a local clipboard cache. | `/plugin install artifacts@danbovey` |

## Layout

```
claude-code-marketplace/
├── .claude-plugin/
│   └── marketplace.json     declares marketplace "danbovey" and lists plugins
└── plugins/
    └── <plugin>/            each plugin is self-contained under here
        ├── .claude-plugin/plugin.json
        ├── bin/             optional executables (added to $PATH)
        ├── skills/<name>/SKILL.md
        └── ...
```

New plugins live as new directories under `plugins/` and a new entry in `.claude-plugin/marketplace.json` — no extra repo per plugin.

## License

[MIT](./LICENSE)
