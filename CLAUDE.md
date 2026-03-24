# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code marketplace (`http4k/agent-skills`) that provides AI coding agent skills for the [http4k](https://http4k.org) Kotlin HTTP toolkit. This is a **documentation and configuration repository** — there is no compiled code, no build system, and no test framework.

## Repository Structure

```
.claude-plugin/
  marketplace.json                 # Marketplace catalog (plugins array, owner)
plugins/http4k/
  .claude-plugin/
    plugin.json                    # Plugin manifest (name, version, author)
  skills/
    http4k-development/            # The single published skill
      SKILL.md                     # Skill definition (detects user's deps)
      references/                  # One file per http4k module
        core.md
        server-undertow.md
        format-jackson.md
        ...
.claude/
  commands/
    generate-skill.md              # /generate-skill command
.github/workflows/                 # CI/CD (version bump on http4k release)
```

### Key Files

- **`.claude-plugin/marketplace.json`** — marketplace catalog listing the http4k plugin
- **`plugins/http4k/.claude-plugin/plugin.json`** — plugin manifest with version tracking http4k releases
- **`plugins/http4k/skills/http4k-development/SKILL.md`** — the published skill; detects user's deps and loads relevant references
- **`.claude/commands/generate-skill.md`** — `/generate-skill` command to regenerate skill content from http4k source

## Version Management

Versions track http4k releases automatically via GitHub Actions. When http4k releases, the `http4k/http4k` repo dispatches an `http4k-release` event which triggers `.github/workflows/update-on-release.yml` to bump the version in `plugins/http4k/.claude-plugin/plugin.json`, commit, tag, and create a GitHub release. **Do not manually change the version.**

## Skill Architecture

There is ONE published skill (`http4k`) with references for ALL http4k modules:

- **`/generate-skill <version>`** clones the http4k repo into `./tmp/http4k` at the specified tag and extracts patterns (primarily from tests). The bootstrap run (`6.25.0.0`) analyzes the full codebase; subsequent runs diff between the previous and new version tags to update only what changed.
- **At runtime**, the published SKILL.md instructs Claude to detect which http4k modules the user's project depends on (via Gradle/Maven build files) and only load the relevant reference files.

## Current State

Phase 1 (Pipeline Skeleton + Marketplace Structure) is complete. Phase 2 will populate `plugins/http4k/skills/http4k-development/` by running `/generate-skill 6.25.0.0`. See `PLAN.md` for full roadmap.

## Branch Note

The workflow pushes to `main` but the repo's default branch is currently `master`. These need alignment before the first automated release.
