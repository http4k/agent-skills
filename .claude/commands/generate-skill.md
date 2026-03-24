---
description: Generate/regenerate the http4k skill from source at a specific version
argument-hint: [version]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git:*), Bash(find:*), Bash(ls:*), Bash(grep:*), Bash(cat:*), Agent
---

Generate or regenerate the http4k skill content from the http4k source code.

## Version Resolution

The version argument is: $ARGUMENTS

If no version was provided, read it from `plugins/http4k/.claude-plugin/plugin.json` and use that version.

## Source Repository

Clone the http4k source into a temporary directory within this repo. Source URL: `https://github.com/http4k/http4k.git`

1. If `./tmp/http4k` already exists, remove it: `rm -rf ./tmp/http4k`
2. Shallow-clone at the specified tag: `git clone --depth 1 --branch $VERSION https://github.com/http4k/http4k.git ./tmp/http4k`
3. If the clone fails (e.g., tag does not exist), report the error and stop

## Tag Comparison Strategy

The version determines the extraction approach:

- **Bootstrap** (no existing skill content at `plugins/http4k/skills/http4k-development/SKILL.md`): Analyze the full codebase at the specified tag in `./tmp/http4k`
- **Update** (skill content already exists): Determine the previous version from `plugins/http4k/.claude-plugin/plugin.json`, then fetch the previous tag into the tmp clone: `git -C ./tmp/http4k fetch --depth 1 origin tag <previous-tag>`. Use `git -C ./tmp/http4k diff <previous-tag>..<new-tag>` to scope the work, but read full files as needed for context

## Module Discovery

Parse the Gradle build files in `./tmp/http4k` to discover ALL published modules. Look for:
- `settings.gradle.kts` for module declarations
- `build.gradle.kts` files for module metadata
- Published module names (e.g., `http4k-core`, `http4k-server-undertow`, `http4k-client-okhttp`, `http4k-format-jackson`, etc.)

## Skill Generation

Generate the **single http4k skill** at `plugins/http4k/skills/http4k-development/`:

### SKILL.md

The main skill file. This is what loads when the skill triggers. It must:

1. Instruct Claude to detect which http4k modules the user's project uses by scanning their Gradle/Maven build files
2. Tell Claude to load `references/{module}.md` for each detected dependency (pattern-matching approach — no need to list files explicitly)
3. Cover core http4k concepts that apply universally (Server as a Function, HttpHandler, Filter, Lens)

Keep lean (1,500–2,000 words).

### references/ (one per module)

Generate a reference file for **every** discovered module (e.g., `references/core.md`, `references/server-undertow.md`, `references/format-jackson.md`, `references/client-okhttp.md`, etc.).

These files are NOT all loaded at once. The SKILL.md instructs Claude to only read references matching the user's project dependencies.

Each reference file should contain:
- Module-specific patterns and configuration
- Gotchas specific to that module
- Testing patterns (fakes, in-memory handlers)
- Code examples extracted from tests

### Extraction Process

Tests are the **primary source** for understanding how to use APIs. They show real usage patterns, construction, configuration, and edge cases. The public API surface tells you *what* exists; tests tell you *how* to use it idiomatically.

For each module:

1. Find the module source under `./tmp/http4k/`
2. **Study test files first** — these are the primary extraction source. Look for idiomatic usage patterns, construction examples, configuration, and edge cases
3. Identify the public API surface (exported classes, functions, extensions) to understand *what* is available
4. Look for `Fake*.kt` or `*Fake.kt` for the testing story — these show how http4k supports testability
5. Search for `companion object` factories to understand construction patterns
6. Find registration/configuration points that are easy to forget (gotchas)
7. Check for `require(` / `check(` calls that indicate common mistakes

### SKILL.md Format

The SKILL.md requires YAML frontmatter:

```markdown
---
name: skill-name
description: This skill should be used when the user asks to "do X", "configure Y", or needs guidance on Z.
---
```

**Frontmatter rules:**
- `name` and `description` are required
- Description must use third person ("This skill should be used when...")
- Include specific trigger phrases users would say

**Body rules:**
- Write in imperative/infinitive form, not second person
- Include correct/wrong code patterns with explanations
- Add implementation checklists for multi-step patterns
- Do NOT list reference files explicitly — the pattern-matching approach (`references/{module}.md`) handles discovery

### Content Conventions

Skills must be:
1. **Self-contained** — all needed context within the skill and its references
2. **Actionable** — concrete patterns with correct/wrong examples
3. **Example-rich** — real http4k code, not abstract descriptions
4. **Gotcha-focused** — prioritize pitfalls that cause runtime failures over obvious patterns
5. **Version-agnostic** — each file is a snapshot in time representing the current state of the API. Describe behavior as it IS, not how it changed.

   **AVOID these patterns:**
   - Temporal/change language: "now defaults to", "new in", "changed from", "was previously", "2x vs 1.x"
   - Bundled/external dependency version numbers that go stale: "Bundles Swagger UI 5.31.0", "HTMX 2.0.8"
   - External product lifecycle dates: "Google ended processing UA hits in July 2023"
   - http4k release version references: "Breaking in 6.32", "new in 6.34", "since 6.26"

   **ACCEPTABLE (not violations):**
   - "deprecated" as current API status (e.g., "`LegacyHttp4kConventions` is deprecated")
   - "Legacy" as a module descriptor
   - "Prefer X for new projects" as guidance
   - External version numbers that are part of module identity (e.g., "Apache HttpClient 4.x" vs "5.x")
   - Semantic convention edition names as API variant identifiers (e.g., "v2"), not pinned spec versions (e.g., "v1.38.0")

### Progressive Disclosure

Follow the three-level loading pattern:

1. **Metadata** (always in context): `name` + `description` from frontmatter (~100 words)
2. **SKILL.md body** (when skill triggers): Core patterns and checklists (<2,000 words)
3. **references/** (loaded as needed): Detailed docs, advanced patterns (unlimited)

Keep SKILL.md lean. Move detailed module documentation, extensive code examples, and advanced patterns into `references/` files.

### Content Priorities

For each reference file, prioritize:

1. **Gotchas** — things that cause runtime failures if forgotten (e.g., value type registration in format modules)
2. **Construction patterns** — idiomatic way to create/configure the module
3. **Testing patterns** — how to test using fakes and in-memory handlers
4. **Common operations** — frequently needed patterns
5. **Anti-patterns** — things that work but are wrong

### New Module Detection (Update runs only)

After computing the diff, explicitly check for new modules:

1. Parse added lines from the diff for `settings.gradle.kts` — look for lines starting with `+include(` to identify newly added modules
2. For each new module name found:
   - Compute the reference file name: strip the `http4k-` prefix → `references/{module}.md`
   - If `references/{module}.md` does NOT exist → treat this module as needing a full reference file generation (same as bootstrap for that module alone)
3. For new modules, perform the full extraction process (study test files, public API, fakes, gotchas) as described in the Bootstrap path
4. Report which new modules were discovered and which reference files will be created

This step runs **in addition to** the diff-based update of existing modules — it handles the case where a new module appears in the release.

## After Generation

1. Verify `plugins/http4k/skills/http4k-development/SKILL.md` has valid frontmatter and includes dependency-detection instructions
2. Verify all generated reference files in `references/` exist
3. Update the version in both manifest files to match the target version:
   - **`plugins/http4k/.claude-plugin/plugin.json`** — set the `version` field
   - **`.claude-plugin/marketplace.json`** — set the `metadata.version` field
4. Report what was generated/updated
5. Clean up: `rm -rf ./tmp/http4k`
