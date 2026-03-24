# http4k Agent Skills — Plan

## Vision

A standalone repository (`http4k/agent-skills`) containing AI coding agent skills for the http4k toolkit. Skills are published to the Claude Code marketplace and versioned in lockstep with http4k releases via an automated pipeline.

## Architecture

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json           # Marketplace catalog (plugins array, owner)
├── plugins/
│   └── http4k/
│       ├── .claude-plugin/
│       │   └── plugin.json        # Plugin manifest (version tracks http4k)
│       └── skills/
│           └── http4k-development/ # Single published skill
│               ├── SKILL.md       # Detects user's deps, loads relevant refs
│               └── references/    # One file per http4k module
├── .claude/
│   └── commands/
│       └── generate-skill.md      # /generate-skill command
├── .github/
│   └── workflows/
│       └── update-on-release.yml  # Receives http4k-release dispatch, bumps version
├── README.md
└── PLAN.md
```

## Release Pipeline

1. http4k releases a new version
2. `http4k/http4k` dispatches `http4k-release` event to `http4k/agent-skills` (via matrix in `CreateUpgradeBranches.kt`)
3. `update-on-release.yml` receives the dispatch:
   - Checks out the repo
   - Updates version in `plugins/http4k/.claude-plugin/plugin.json`
   - Commits, tags, and creates a GitHub release
4. Claude Code marketplace picks up the new release

## Phases

### Phase 1: Pipeline Skeleton + Marketplace Structure ✅
- [x] Create `marketplace.json` with marketplace catalog format (plugins array, owner)
- [x] Create `plugins/http4k/.claude-plugin/plugin.json` with plugin manifest
- [x] Create `update-on-release.yml` workflow (targets plugin.json for version bumps)
- [x] Update README and CLAUDE.md to reflect structure
- [x] Wire http4k dispatch to include `http4k/agent-skills`
- [x] Delete old `claude-skills/` directory from http4k repo

### Phase 2: Skill Content Generation (`/generate-skill 6.25.0.0`) ✅

Bootstrap run complete for v6.25.0.0. All reference files generated.

Excluded from generation: `bom`, `ksp-generator`, `example`, integration tests, test functions, `shutdown-integration-test`.

#### Batch 1 — Core & Config
- [x] `core` — HttpHandler, Filter, Lens, Routing, Events, SunHttp
- [x] `config` — Environment, configuration
- [x] `multipart` — Multipart form handling
- [x] `realtime-core` — WebSocket/SSE core

#### Batch 2 — Servers
- [x] `server-undertow`
- [x] `server-jetty`
- [x] `server-jetty11`
- [x] `server-netty`
- [x] `server-apache`
- [x] `server-apache4`
- [x] `server-helidon`
- [x] `server-ktorcio`
- [x] `server-ktornetty`
- [x] `server-ratpack`
- [x] `server-websocket`

#### Batch 3 — Clients
- [x] `client-okhttp`
- [x] `client-apache`
- [x] `client-apache-async`
- [x] `client-apache4`
- [x] `client-apache4-async`
- [x] `client-fuel`
- [x] `client-helidon`
- [x] `client-jetty`
- [x] `client-websocket`

#### Batch 4 — Format Modules
- [x] `format-core`
- [x] `format-jackson`
- [x] `format-jackson-xml`
- [x] `format-jackson-yaml`
- [x] `format-jackson-csv`
- [x] `format-gson`
- [x] `format-moshi`
- [x] `format-moshi-yaml`
- [x] `format-kotlinx-serialization`
- [x] `format-kondor-json`
- [x] `format-klaxon`
- [x] `format-argo`
- [x] `format-dataframe`
- [x] `format-toon`
- [x] `format-xml`

#### Batch 5 — API Modules
- [x] `api-openapi`
- [x] `api-graphql`
- [x] `api-jsonrpc`
- [x] `api-jsonschema`
- [x] `api-cloudevents`
- [x] `api-ui-swagger`
- [x] `api-ui-redoc`

#### Batch 6 — Security
- [x] `security-core`
- [x] `security-digest`
- [x] `security-oauth`

#### Batch 7 — Testing
- [x] `testing-approval`
- [x] `testing-hamkrest`
- [x] `testing-kotest`
- [x] `testing-strikt`
- [x] `testing-powerassert`
- [x] `testing-chaos`
- [x] `testing-servirtium`
- [x] `testing-tracerbullet`
- [x] `testing-webdriver`
- [x] `testing-playwright`

#### Batch 8 — Templates
- [x] `template-core`
- [x] `template-handlebars`
- [x] `template-thymeleaf`
- [x] `template-pebble`
- [x] `template-freemarker`
- [x] `template-jte`
- [x] `template-rocker`
- [x] `template-pug4j`
- [x] `template-htmlflow`

#### Batch 9 — Ops & Observability
- [x] `ops-core`
- [x] `ops-micrometer`
- [x] `ops-opentelemetry`
- [x] `ops-resilience4j`
- [x] `ops-failsafe`

#### Batch 10 — Platform & Serverless
- [x] `platform-core`
- [x] `platform-aws`
- [x] `platform-azure`
- [x] `platform-gcp`
- [x] `platform-k8s`
- [x] `platform-docker`
- [x] `serverless-core`
- [x] `serverless-lambda`
- [x] `serverless-lambda-runtime`
- [x] `serverless-gcf`
- [x] `serverless-azure`
- [x] `serverless-alibaba`
- [x] `serverless-openwhisk`
- [x] `serverless-tencent`

#### Batch 11 — Bridges
- [x] `bridge-servlet`
- [x] `bridge-spring`
- [x] `bridge-ktor`
- [x] `bridge-helidon`
- [x] `bridge-jakarta`
- [x] `bridge-micronaut`
- [x] `bridge-ratpack`
- [x] `bridge-vertx`

#### Batch 12 — Web
- [x] `web-htmx`
- [x] `web-datastar`
- [x] `webhook`

#### Batch 13 — Tools
- [x] `tools`
- [x] `tools-hotreload`
- [x] `tools-traffic-capture`
- [x] `incubator`

#### Batch 14 — AI & MCP
- [x] `ai-core`
- [x] `ai-llm-core`
- [x] `ai-llm-openai`
- [x] `ai-llm-anthropic`
- [x] `ai-llm-azure`
- [x] `ai-llm-gemini`
- [x] `ai-llm-github`
- [x] `ai-langchain4j`
- [x] `ai-mcp-core`
- [x] `ai-mcp-sdk`
- [x] `ai-mcp-client`
- [x] `ai-mcp-conformance`

#### Batch 15 — Connect Core & Storage
- [x] `connect-core`
- [x] `connect-core-fake`
- [x] `connect-storage-core`
- [x] `connect-storage-http`
- [x] `connect-storage-jdbc`
- [x] `connect-storage-redis`
- [x] `connect-storage-s3`

#### Batch 16 — Connect AI
- [x] `connect-ai-openai` + fake
- [x] `connect-ai-anthropic` + fake
- [x] `connect-ai-azure` + fake
- [x] `connect-ai-ollama` + fake
- [x] `connect-ai-lmstudio` + fake
- [x] `connect-ai-langchain`

#### Batch 17 — Connect Amazon (Part 1)
- [x] `connect-amazon-core`
- [x] `connect-amazon-s3` + fake
- [x] `connect-amazon-dynamodb` + fake
- [x] `connect-amazon-sqs` + fake
- [x] `connect-amazon-sns` + fake
- [x] `connect-amazon-lambda` + fake
- [x] `connect-amazon-sts` + fake
- [x] `connect-amazon-kms` + fake

#### Batch 18 — Connect Amazon (Part 2)
- [x] `connect-amazon-cognito` + fake
- [x] `connect-amazon-secretsmanager` + fake
- [x] `connect-amazon-systemsmanager` + fake
- [x] `connect-amazon-ses` + fake
- [x] `connect-amazon-cloudwatch` + fake
- [x] `connect-amazon-cloudwatchlogs` + fake
- [x] `connect-amazon-cloudfront` + fake
- [x] `connect-amazon-eventbridge` + fake
- [x] `connect-amazon-evidently` + fake
- [x] `connect-amazon-firehose` + fake
- [x] `connect-amazon-apprunner` + fake
- [x] `connect-amazon-route53` + fake
- [x] `connect-amazon-scheduler` + fake
- [x] `connect-amazon-iamidentitycenter` + fake
- [x] `connect-amazon-instancemetadata` + fake
- [x] `connect-amazon-containercredentials` + fake

#### Batch 19 — Connect Third-Party
- [x] `connect-github` + fake
- [x] `connect-gitlab` + fake
- [x] `connect-slack` + fake
- [x] `connect-mattermost` + fake
- [x] `connect-kafka-rest` + fake
- [x] `connect-kafka-schemaregistry` + fake
- [x] `connect-google-analytics-core`
- [x] `connect-google-analytics-ga4` + fake
- [x] `connect-google-analytics-ua` + fake

#### After All Batches
- [x] Update SKILL.md if needed
- [x] Final verification pass
- [x] `rm -rf ./tmp/http4k`

### Phase 3: Incremental Updates
- Subsequent `/generate-skill <version>` runs diff between previous and new version tags of the http4k source code
- Only updates reference files for modules that changed between versions

