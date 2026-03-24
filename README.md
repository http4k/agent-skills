# http4k Agent Skills

AI coding agent skills for the [http4k](https://http4k.org) Kotlin HTTP toolkit.

## Overview

This repository is a Claude Code marketplace providing skills that teach AI coding agents how to work effectively with http4k. Skills include reference documentation, coding patterns, and anti-patterns for each http4k module.

## Version

Skill versions track http4k releases automatically. When a new http4k version is released, this repository is updated via GitHub Actions dispatch.

## Structure

```
.claude-plugin/
  marketplace.json               # Marketplace catalog
plugins/http4k/
  .claude-plugin/
    plugin.json                  # Plugin manifest + version
  skills/
    http4k-development/          # The single published skill
      SKILL.md                   # Skill definition (detects user's deps)
      references/                # One reference file per http4k module
        core.md
        server-undertow.md
        format-jackson.md
        ...
```

When activated, the skill scans the user's project build files (Gradle/Maven) to detect which http4k modules are in use, then loads only the relevant reference files.

## License

See [http4k.org](https://http4k.org) for licensing information.
