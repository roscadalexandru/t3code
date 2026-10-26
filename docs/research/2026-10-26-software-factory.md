# Software factory features for T3 Code: Factory, Warp, T3 today and the user's factory

Status: research, done on 2026-10-26. Read-only: no code was changed.

All web sources were read on 2026-10-26. This document builds on two earlier studies in
factory-hub: `docs/research/2026-10-24-factory-com.md` (Factory) and
`docs/research/2026-10-22-one-factory.md`, section 6 (Warp and comparable systems).
Section 4.3 and section 5.3 re-check their claims.

## 0. Versions and how to read the citations

- The running T3 Code is `0.1.3-nightly.20261023.3566`
  (`~/.t3/runtime/versions/0.1.3-nightly.20261023.3566/`).
- Its source is the upstream tag `v0.1.3-nightly.20261023.3566` of
  `github.com/pingdotgg/t3code`, commit `807319c276c6`, dated 2026-10-23.
- **Every T3 `path:line` below refers to that upstream tag, not to this checkout.**
- When this research started, the fork `roscadalexandru/t3code` was on `main` at
  `24c4ba68f`, dated 2026-08-19. That `main` had no orchestrator MCP toolkit and no
  `apps/server/src/scheduledTasks/`.
- Update on 2026-10-26: the fork's `main` had no commits that upstream lacked. It was
  fast-forwarded to upstream `main` at `a240408242` (2026-10-26) and pushed. The fork now has
  every T3 feature that this document cites. Step 0 (section 9.3) is done.
- The `path:line` citations still refer to the upstream tag. Lines can differ slightly in the
  fork's newer `main`. For example, the `baseRef: "main"` default is at
  `apps/server/src/mcp/OrchestratorMcpService.ts:289` in `a240408242`.
- Paths that start with `factory-hub/` refer to `/home/admin/factory-hub`.
- "Marketing only" means that the claim is on a marketing page, not in product docs.
- "Not verified" means that I could not confirm the claim in a primary source or in code.

## 1. The question

The user asked on 2026-10-26:

1. Which capabilities make Factory (factory.com, "Droids") and Warp a "software factory"?
   List each one with its first-party source.
2. For each capability, what does T3 Code already have natively?
3. What does the user's factory (`/home/admin/factory-hub`) do? Which parts are
   provider-agnostic, so they are good candidates to move into T3? Which parts are specific
   to its Claude Code plugin packaging?
4. A gap matrix (capability × Factory, Warp, T3 today, the user's factory), and the smallest
   ordered set of native T3 features.
5. The open questions that only the user can answer.

Constraint from the user: no Factory Droid provider. The factory features must be native in
T3, so that they work with every existing provider.

## 2. Terms

| Term | Meaning in this document |
| ---- | ------------------------ |
| software factory | A system in which agents turn work items into reviewed, merged pull requests (PRs), with few human steps. |
| job | One work item that goes through the loop, from claim to merge or stop. This is the factory-hub term. |
| loop | The ordered stages that a job goes through. |
| stage | One step of the loop, for example spec, build, code review, approve or merge queue. |
| role | A named kind of agent with its own instructions, for example builder or security reviewer. |
| gate | A point where the loop waits for a human decision. |
| trigger | An event that starts a run: a schedule, a webhook, an issue, or a chat message. |
| orchestrator | The agent that runs the loop and starts the role agents. |
| child thread | A T3 thread that `delegate_task` creates for a role. |
| Factory | factory.com, the company that sells Droids. Its old name was factory.ai. |
| Droid | Factory's agent harness: CLI, desktop app, and `droid exec`. |
| Warp Factories | Warp's software-factory product (section 5.1). |
| foreman | The orchestrator agent in Warp Factories. |
| the user's factory | factory-hub: the Claude Code plugin `factory`, the engine CLI `factory`, and the hub. |
| T3 | T3 Code. The terms provider, adapter, client, environment, project, thread and turn have the meanings in `AGENTS.md`. |

## 3. Answer in brief

- **"Warp factories" is Warp's product "Warp Factories"** (<https://docs.warp.dev/factories/>).
  Warp launched it on 2026-08-18. The concept post is "The factory stack" (2026-09-05).
- **Factory and Warp have the same shape.** Triggers bring work in. An orchestrator runs role
  agents through stages. Humans act at gates. A dashboard measures cost, time and autonomy.
- **Factory** has Missions, custom droids, Spec Mode, autonomy levels, Automations, delegation
  from Slack, Linear, Jira and Teams, automated PR review, hosted compute, a model router, and
  analytics. Several parts are Private Preview. The cloud features need GitHub or GitLab.
- **Warp Factories** is in Early Access. A `factory.yaml` tree defines agents, triggers,
  routers and scorers. A foreman runs the stages. A personal inbox and a dashboard show
  autonomy, cycle time and cost per PR. "The factory never merges on your behalf."
- **T3 already has most of the provider-neutral building blocks.** These are schedule and
  webhook triggers, worktree isolation, cross-provider `delegate_task`, PR watch (also on
  Gitea), an inbox, push notifications, a headless CLI, and an MCP endpoint for outside agents.
  They work with every adapter that has `supportsMcpTools` (section 6.1).
- **T3 lacks five things that a factory needs.** These are committed role definitions, a
  durable gate state with a reason, per-job measures, native tracker or chat triggers, and
  scorers. T3 also has no hosted compute, and none is proposed.
- **The user's factory fills the first three gaps outside T3.** It is Claude-specific in five
  places (section 7.3). Its loop, rules, engine and hub are provider-agnostic.
- **One small T3 defect blocks the user's factory now.** The MCP `schedule_task` hard-codes
  `baseRef: "main"` (`apps/server/src/mcp/OrchestratorMcpService.ts:285-291`). The contract,
  the web UI and the mobile UI all support another base branch.
- **Recommendation (section 9, kept apart from the facts):** rebase the fork (done on
  2026-10-26), port the run
  skill to T3 MCP tools without a T3 change, then fix the MCP base branch and hook path. After
  that, add committed agent presets only if the port shows the need, then a durable gate, then
  a per-thread measure read. Do not build a workflow engine, a Droid provider or native
  Slack, Linear or Jira integrations now.

## 4. Factory (factory.com)

### 4.1 Status, domains and vocabulary

- `factory.ai` redirects to `factory.com`. `docs.factory.ai` redirects to `docs.factory.com`.
  The app, API and CLI hosts still use factory.ai, for example `app.factory.ai`.
- Factory uses two maturity tags: "Private Preview" and "Deprecated". A page without a tag is
  generally available (<https://docs.factory.com/changelog/feature-maturity>).
- The newest changelog entry is CLI v0.249.0 and Desktop v0.206.0, dated 2026-10-24
  (<https://docs.factory.com/changelog/release-notes>).
- The words "software factory" are mostly marketing. The home page says "BUILD YOUR
  AUTONOMOUS SOFTWARE FACTORY" (<https://factory.com/>, marketing only).
- The docs say: "Software Factory is a 24/7 autonomous system that connects AI agents across
  the software development lifecycle". That page is Private Preview
  (<https://docs.factory.com/software-factory/overview>).
- The docs group the Software Factory into stages: Triage, Code-gen, Validate, Release,
  Document and Monitor (same page).

### 4.2 Capabilities

The IDs (F1, F2, ...) are used again in the gap matrix (section 8).

| ID | Capability | Evidence | Maturity |
| -- | ---------- | -------- | -------- |
| F1 | Headless runs | "`droid exec` runs as a one-shot command that completes a task and exits, making it ideal for CI/CD pipelines, shell scripts, and batch processing" (<https://docs.factory.com/droid-exec/overview>). Output formats: `text`, `json`, `stream-json`, `stream-jsonrpc`, `acp`. "By default the run is read-only". | GA |
| F2 | SDKs and API | Python and TypeScript SDKs "run the same agent harness that powers Factory's CLI, desktop application, and web platform" (<https://docs.factory.com/sdk/python>). The Sessions API is "enabled for selected organizations only" (<https://docs.factory.com/api-reference/sessions>). | GA; API limited |
| F3 | Role agents (custom droids) | "Each droid carries its own system prompt, model preference, and tool policy". "When a project droid and a personal droid share the same name, the project definition wins." Built-in droids: `worker` and `explorer`. Subagents cannot start subagents. (<https://docs.factory.com/harness/subagents>) | GA |
| F4 | Extensions as files | Skills, plugins, hooks (9 events), MCP and `AGENTS.md`. "Claude Code plugin layouts are also supported." (<https://docs.factory.com/harness/plugins>, <https://docs.factory.com/harness/hooks>, <https://docs.factory.com/harness/skills>) | GA |
| F5 | Multi-agent orchestration (Missions) | An orchestrator runs workers in milestones. "Validation workers run at the end of each milestone, verifying its work." (<https://docs.factory.com/missions/planning>). Per-role models: `--worker-model`, `--validator-model`. "Missions are still evolving." (<https://docs.factory.com/missions/overview>). Missions "require Extra Usage to be enabled" on Individual plans (<https://docs.factory.com/pricing/individuals>). | GA, "evolving" |
| F6 | Spec mode with approval | Spec Mode "Uses read-only planning behavior, then calls `ExitSpecMode` to ask for approval." Modes: Normal, Spec, Mission. (<https://docs.factory.com/autonomy-and-safety/specification-mode>). `droid exec --use-spec`. | GA |
| F7 | Autonomy levels and permission rules | "Autonomy Level sets the highest-risk work Droid can run without pausing for approval." Levels: Off, Low, Medium, High (<https://docs.factory.com/autonomy-and-safety/auto-run>). Rules allow, ask or block; "`ask` requires it even at High" (<https://docs.factory.com/autonomy-and-safety/permission-rules>). Admins cap it with `maxAutonomyLevel`. | GA |
| F8 | Triggers: schedule, Slack, GitHub, webhook | "What starts a run: a schedule, a Slack message, a GitHub event, or an HTTP `POST` to the automation's webhook URL." "Webhook automations can't run on your own machine." (<https://docs.factory.com/software-factory/automations>) | GA; webhooks Private Preview |
| F9 | Triggers: trackers and chat (Delegations) | "Delegations lets your team send work to Droid from Slack, Linear, Jira, and Microsoft Teams." (<https://docs.factory.com/delegations>) | GA |
| F10 | Ready-made automations | Guided setups: Triage, PR Babysitter, Incident Response, Code Review, QA, AutoWiki, Security Audit (<https://docs.factory.com/software-factory/automations>). Release is a stage that tracks gates (<https://docs.factory.com/software-factory/release>, Private Preview). | Mixed |
| F11 | Automated PR review | "Triggers on pull request events (opened, synchronize, reopened, ready for review)". It skips drafts and "Submits an approval when no issues are found". It runs as `Factory-AI/droid-action` on GitHub Actions or GitLab CI (<https://docs.factory.com/software-factory/code-review-ci>). | GA |
| F12 | Worktrees | "Factory-managed worktrees are available for Git projects on your local machine and connected Droid Computers." Setup profiles: `.factory/worktree-setups/*.yaml` (<https://docs.factory.com/factory-app/worktrees>). `droid exec -w`. | GA |
| F13 | Hosted compute | Droid Computers: "Persistent compute environments hosted and managed by Factory for Droid." (<https://docs.factory.com/droid-computers/overview>). Environments are GA since 2026-10-12 and replace Cloud Templates (<https://docs.factory.com/environments/overview>). OpenCompute is a self-hosted orchestrator (<https://docs.factory.com/opencompute/overview>). | GA; OpenCompute Private Preview |
| F14 | Model choice | "Factory Router automatically chooses the best model for each Droid task." (<https://docs.factory.com/model-independence/factory-router>). BYOK: "Use your own OpenAI or Anthropic keys, connect to any open source model providers, or run models locally" (<https://docs.factory.com/model-independence/byok>). Models from Anthropic, OpenAI, Google, xAI and open models (<https://docs.factory.com/models>). | GA |
| F15 | Dashboards and measures | "Measure Droid usage with the hosted Analytics dashboards" (<https://docs.factory.com/agent-effectiveness/cost-and-productivity>). Agent Effectiveness "connects agent spend to the work it produces" (Private Preview). Mission Control. An Automations dashboard with an events feed and a 7-day success rate. | Mixed |
| F16 | Repo readiness | "Repositories progress through five distinct levels". `/readiness-report`, `/readiness-fix`, a dashboard and an API (<https://docs.factory.com/agent-readiness/overview>). | GA |
| F17 | Org knowledge | `AGENTS.md`. AutoWiki: "Run `/install-wiki` to refresh Wikis on every push" (<https://docs.factory.com/software-factory/wiki/auto-refresh>). Scheduled automations have "Memory & Evolution". A separate memory or knowledge-base feature was not found. | GA; memory not verified |
| F18 | Secret scan before push | Droid Shield "automatically scans un-committed changes for potential secrets before committing and pushing them to remote." Droid Shield 2.0 is Private Preview (<https://docs.factory.com/autonomy-and-safety/droid-shield>). | GA |
| F19 | Session sharing and remote control | "every CLI session is mirrored to Factory web" (<https://docs.factory.com/droid-cli/settings>). `/share`, `/fork`, `/tree`. "Connect your computer so Droid can work on it from the web or other devices." (<https://docs.factory.com/remote-control>). Mobile is the browser. A native mobile app was not found. | GA; mobile app not verified |
| F20 | Git hosts | Code review: "GitHub or GitLab repositories" (<https://docs.factory.com/software-factory/code-review-ci>). Self-managed SCM: "a self-hosted GitHub Enterprise Server or GitLab instance", Enterprise plan (<https://docs.factory.com/enterprise/self-managed-scm>). No page says "only". | GA |
| F21 | Enterprise access | "SSO (SAML/OIDC), and Directory Sync (SCIM)" (<https://docs.factory.com/enterprise/identity-and-access>). Audit log: "available to **Enterprise organization owners only**" (<https://docs.factory.com/enterprise/audit-log>). The pricing pages put audit logging under Business: the sources conflict. | GA |
| F22 | Operating software | "Droid Control lets Droids *operate* software: launch apps, type commands, click buttons, record what happens" (<https://docs.factory.com/software-factory/droid-control>). | GA |

### 4.3 Re-check of the 2026-10-24 claims

The claims are the rows of `factory-hub/docs/research/2026-10-24-factory-com.md:21-32` and
`:106-129`.

| # | Claim in 2026-10-24 | Result | What changed or is new |
| - | ------------------- | ------ | ---------------------- |
| 1 | Droid CLI and `droid exec` for headless runs | Confirmed (F1) | New: `--auto low\|medium\|high`, `--use-spec`, `--mission`, `-w`, tool allow-lists, and an `acp` output format. |
| 2 | Factory App runs sessions in worktrees | Confirmed (F12) | New: shared setup profiles. Caveat: the welcome page offers Mac and Windows downloads. A changelog entry (Oct 20) mentions "Linux in-app updates". A Linux build is not verified. |
| 3 | Custom droids with own prompt, model and tools | Confirmed (F3) | Changed: the docs name project and personal droids only. The earlier "project over shared" wording (`2026-10-22-one-factory.md:274`) has no "shared" level in the docs. |
| 4 | Skills, plugins, hooks, MCP, `AGENTS.md`; loads Claude Code layouts | Confirmed (F4) | New: skills also load from `.agents/skills`. |
| 5 | Missions with orchestrator, workers, validators | Confirmed (F5) | New: per-role model flags. The orchestrator view is "Mission Control". |
| 6 | Automations on schedule, Slack, GitHub, webhook | Confirmed (F8) | Changed: webhook triggers are Private Preview. They run on a Droid Computer or an Environment. A news post says "Automations are generally available in Factory" (<https://factory.com/news/automations>, marketing only). |
| 7 | Software Factory: ready automations | Changed (F10) | The docs now group stages: Triage, Code-gen, Validate, Release, Document, Monitor. The overview is Private Preview. New: Local Code Review (`/review`) and Droid Control. |
| 8 | Droid Computers | Confirmed, sources conflict (F13) | The home page says "Purpose-built hardware that runs your agents locally" (marketing only). This contradicts the docs. Pricing puts Droid Computers in Plus. The docs put them in Pro. |
| 9 | Factory Router and BYOK | Confirmed (F14) | Also labelled "Auto Model". Factory claims 63% cost savings (marketing only). |
| 10 | Analytics and Agent Readiness | Confirmed (F15, F16) | Analytics is for "Business and Enterprise plans" (<https://factory.com/news/factory-analytics-transparency>, marketing only). |
| 11 | Slack, Linear, Jira and Teams delegation | Confirmed (F9) | None. |
| 12 | Droid Shield | Confirmed (F18) | New: Droid Shield 2.0 (Private Preview). |
| 13 | Cloud features only on GitHub, GHE and GitLab | Confirmed by inference (F20) | No page says "only". Exception: internal plugin marketplaces mention "Connected GitLab, Bitbucket, and Azure DevOps integrations", for git credentials on Droid Computers (<https://docs.factory.com/enterprise/internal-plugin-marketplaces>). |
| 14 | Enterprise: SSO, ZDR, audit log, on-premise | Confirmed, sources conflict (F21) | Audit log plan conflict. New: "Factory Private" in "their own VPCs, on-premises environments, or air-gapped networks" (<https://factory.com/news/factory-private>, Private Preview, marketing only). |
| - | "T3 Code has no Droid provider" (`2026-10-24-factory-com.md:114`) | Still true at the upstream tag | `BUILT_IN_DRIVERS` lists Codex, Claude, Cursor, Grok, OpenCode, Antigravity, Pi, Muse and AcpRegistry (`apps/server/src/provider/builtInDrivers.ts:29-39`). Droid lists an `acp` output format. Whether Droid runs through T3's generic ACP driver is not verified. The user does not want this path. |

## 5. Warp

### 5.1 What "warp factories" refers to

- The product is **Warp Factories**. "Warp Factories turns issues, tickets, and Slack messages
  into reviewed pull requests. Teams of cloud agents do the work, while people approve specs
  and merges." (<https://docs.warp.dev/factories/>). The web app is `platform.warp.dev`.
- The launch post is "Introducing Warp Factories", Zach Lloyd, 2026-08-18
  (<https://www.warp.dev/blog/open-infrastructure-for-building-a-software-factory>).
- The concept post is "The factory stack", 2026-09-05. It names seven layers:
  factories-as-code, data and context, compute, inference, improvement, orchestration and
  access. "Software factories should be built on an infrastructure stack that is open,
  composable and defined in code."
- Warp Factories runs on Warp's Automation Platform
  (<https://docs.warp.dev/platform/overview/>). "Oz" is the older name for Warp's standalone
  cloud agents (<https://docs.warp.dev/platform/transitioning-from-oz/>). The Warp Agent
  harness has the id `oz`.
- Status: the docs navigation shows an "Early Access" badge. The page body does not state a
  status. The product is not generally available. The GA date is not verified.

### 5.2 Capabilities

| ID | Capability | Evidence (docs.warp.dev unless noted) |
| -- | ---------- | -------------------------------------- |
| W1 | Factory as code | "A factory definition is a versioned tree of YAML and Markdown files rooted at `factory.yaml`. It can define agents, automations, runners, model routers, benchmarks, scorers, skills, and webhooks." `agentDefaults` must "Declare exactly one of model or harness" (<https://docs.warp.dev/factories/factory-as-code/>). Warp keeps the definition by default; GitHub needs a paid plan. A `warp/factory-config` PR check validates it. JSON Schemas `v1alpha1`. Examples: <https://github.com/warpdotdev/warp-factory-examples> (MIT). |
| W2 | Role agents | A foreman plus up to four default agents: triage, spec, implement and review. `agentType` also allows `CUSTOM` and `VERIFY`. The foreman dispatches with `run_agents`. The review agent "doesn't approve or merge" (<https://docs.warp.dev/factories/factory-agents/>). |
| W3 | Orchestrated stages | Stages: "Intake", "Triage", "Planning", "Building", "Reviewing", "Human handoff", "Complete or Cancelled". "The foreman picks the shortest path that still meets your quality policy" (<https://docs.warp.dev/factories/how-factories-work/>). |
| W4 | Spec approval and question gates | "When work needs a spec, implementation waits until a person signs off on the plan." "When requirements are unclear or a review finding is ambiguous, the foreman asks instead of guessing." (same page). These gates "are workflow policy, written into the foreman's instructions". |
| W5 | Merge policy | "Merging is enforced by your repository" (how-factories-work). "The factory never merges on your behalf." (<https://docs.warp.dev/factories/>). Conflict: the 2026-08-18 launch post implies a direct merge is possible. |
| W6 | Triggers | "Route work into your factory from Slack, Microsoft Teams, GitHub, GitLab, Azure DevOps, Linear, Jira, local agents, direct runs, and schedules." Also generic webhooks and a `work_item_stage_changed` event. Signature verifiers: `github`, `pagerduty`, `sentry`, `standard_webhooks`, `stripe`, `vercel`. A factory cannot use Linear and Jira at once. Teams and Azure DevOps came on 2026-10-07 (<https://docs.warp.dev/factories/connect-your-factory/>). |
| W7 | API, SDK and MCP | `POST /factory/{uid}/runs`, Python and TypeScript SDKs (<https://docs.warp.dev/factories/factory-api/>). Factory MCP at `app.warp.dev/api/v1/mcp/factory` for Claude Code, Cursor, Codex and others. "Factory MCP has no read-only or per-factory scopes" (<https://docs.warp.dev/factories/factory-mcp/>). No factory CLI was found. |
| W8 | Hosted compute | Warp-hosted sandboxes for Linux, macOS and Windows. "Managed self-hosting requires an Enterprise plan" (Docker, Kubernetes or Direct). "Unmanaged self-hosted agents and other CLI agents can't serve as a factory's execution host" (<https://docs.warp.dev/factories/infrastructure-and-security/>, <https://docs.warp.dev/factories/runners/>). |
| W9 | Harnesses and models | "Supported harnesses include the Warp Agent, Claude Code, and Codex." "Third-party harnesses require a paid plan." (<https://docs.warp.dev/factories/factory-agents/configuration/>). Custom routers use "one of two routing types": complexity-based ("easy, medium, or hard") or rule-based (<https://docs.warp.dev/agents/inference/custom-routers/>). Inference is Warp-managed, team keys (Business and up) or Bedrock (Enterprise). |
| W10 | Inbox | One personal inbox across factories. "Inbox only shows work tied to tasks you started". Six item types: asked question, answered question, spec review, PR review, blocked, failed (<https://docs.warp.dev/factories/factory-inbox/>). |
| W11 | Dashboard | Autonomy is the "share of the factory's merged PRs that needed no human code push before merging". Also PR cycle time (median and per stage), cost per PR and runs. The metrics "require a connected code host" (<https://docs.warp.dev/factories/factory-dashboard/>). |
| W12 | Scorers and self-improvement | LLM-judge scorers and benchmarks. Self-improvement runs after 25 unreviewed failures or 7 days and opens PRs. "Nothing is adopted without your review" (<https://docs.warp.dev/factories/>, improvement pages). |
| W13 | Skills and memory | Skills as `SKILL.md`. Agent Memory (<https://docs.warp.dev/agents/agent-memory/>) is a research preview and is not tied to factories. Memory as a factory feature is marketing only. |
| W14 | Access | Roles: User, Editor, Admin. "Factory roles don't control specification or pull request review." `credentialStrategy`: `EXECUTOR` or `CREATOR` (paid). |
| W15 | Multi-repo | "Group the repositories that ship together into one factory, and keep separate products in separate factories." (<https://docs.warp.dev/factories/>). Runs, MCPs, secrets and integrations are shared across factories. No templates or inheritance were found. |
| W16 | Pricing | Pay as you go $0 (+20% over API rates), Build $20/mo (Claude Code and Codex harnesses, GitHub definition), Business $50/user/mo, Enterprise custom (<https://www.warp.dev/pricing>, marketing only). The docs bill per agent hour. The pricing FAQ says per agent run: the sources conflict. |

### 5.3 Re-check of the 2026-10-22 claims

The claims are in `factory-hub/docs/research/2026-10-22-one-factory.md:193-216` and the Warp
row at `:268`.

| Claim | Result | What changed or is new |
| ----- | ------ | ---------------------- |
| "The factory stack" has seven layers and says "open, composable and defined in code" | Confirmed | None. |
| A foreman routes work through intake, triage, planning, building, reviewing and human handoff; it can skip a stage | Confirmed (W3) | The final stage is "Complete or Cancelled". The foreman completes the work item after the merge notification. |
| `factory.yaml` has `schemaVersion`, `name`, `repositories`, `agentDefaults`; agents override defaults | Confirmed, broader (W1) | The tree can also define runners, model routers, benchmarks, scorers, skills and webhooks. A PR check validates it. The last valid definition stays active. |
| The dashboard shows autonomy, PR cycle time, cost per PR, runs and an inbox | Confirmed, refined (W10, W11) | Autonomy counts merged PRs with no human code push. The inbox is personal and has six item types. Metrics need a connected code host. |
| Merges are governed by the repo | Confirmed (W5) | "The factory never merges on your behalf." The launch post conflicts. |
| New since 2026-10-22 | New | Triggers (W6), Factory MCP (W7), hosted compute only (W8), Claude Code and Codex harnesses (W9), scorers and self-improvement (W12), pricing (W16). Changelog 2026.09.09: "Claude Code and Codex runs can now access Factory MCP automatically" (<https://docs.warp.dev/_llms-txt/changelog.txt>). |

## 6. T3 Code today

### 6.1 Providers and the decision per adapter

`BUILT_IN_DRIVERS` lists Codex, Claude, Cursor, Grok, OpenCode, Antigravity, Pi, Muse and
AcpRegistry (`apps/server/src/provider/builtInDrivers.ts:29-39`). Each adapter declares its
capabilities in the contract (`packages/contracts/src/orchestrationV2.ts:210-347`).

| Adapter | T3 MCP tools | Proposed plan | Structured questions | Command approval | Source |
| ------- | ------------ | ------------- | -------------------- | ---------------- | ------ |
| Claude | yes | yes | yes | yes | `apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts:394-414` |
| Codex | yes | yes | yes | yes | `apps/server/src/orchestration-v2/Adapters/CodexAdapterV2.ts:277-297` |
| Cursor | yes | yes | **no** | **no** | `packages/provider-cursor/src/server/adapter.ts:124-144` |
| Grok | yes | no (ACP default) | yes (ACP default) | yes (ACP default) | `packages/provider-grok/src/server/adapter.ts:95-117`, defaults at `packages/provider-acp/src/server/adapter.ts:510-530` |
| Antigravity | yes | no (ACP default) | yes (ACP default) | yes (ACP default) | `apps/server/src/orchestration-v2/Adapters/AntigravityAdapterV2.ts:45-60` |
| OpenCode | yes on the v1 runtime, **no on the v2 runtime** | no | yes | yes | `packages/provider-opencode/src/server/adapter.ts:157-177`, `packages/provider-opencode/src/server/v2/adapter.ts:131-151`; a runtime probe picks one (`packages/provider-opencode/src/server/driver.ts:287-292`) |
| Pi | yes | no | yes | yes | `packages/provider-pi/src/server/adapter.ts:137-157` |
| Muse | yes | no | yes | yes | `packages/provider-muse/src/server/adapter.ts:139-159` |
| Generic ACP | no at start; yes after the session negotiates | no | yes | yes | `packages/provider-acp/src/server/adapter.ts:510-530`, `:568-593` |

- Every adapter except generic ACP declares `supportsSubagents: true` (same files).
- T3 injects shared orchestration instructions into every provider. They cover
  `delegate_task` against native subagents, `schedule_task`, `request_secret` and the
  `t3_thread_launch` workspace strategy (`packages/provider-core/src/server/orchestrationInstructions.ts:8`).
- An ACP agent without MCP tools gets a fallback through the `acp-mcp-call` CLI command
  (`apps/server/src/cli/acpMcpBridge.ts:15`).
- Runtime modes are `approval-required`, `auto-accept-edits`, `auto` and `full-access`. The
  default is `full-access`. The interaction mode is `default` or `plan`
  (`packages/contracts/src/providerPolicy.ts:25-36`).
- **Consequence for a factory:** an orchestrator thread needs T3 MCP tools. It cannot run on
  OpenCode's v2 runtime or on a generic ACP agent before negotiation. A role agent that only
  edits files and returns text can run on any adapter.

### 6.2 Triggers: scheduled tasks and webhooks

- Schedules: `interval` (at least 60 000 ms) and `fixed_time` (wall-clock time on chosen
  weekdays) (`packages/contracts/src/scheduledTask.ts:26-45`).
- Webhooks: an HMAC-SHA256 signature check with a configurable header, hex or base64, and a
  prefix (`packages/contracts/src/scheduledTask.ts:59-73`). Filters: `equals`, `notEquals`,
  `startsWith`, `in`, `exists`, up to 16 (`:106-146`).
- Gitea signs with `X-Gitea-Signature` as hex without a prefix. The configurable header fits
  this. A real Gitea delivery to T3 is not verified.
- A task carries `workspaceStrategy`, `modelSelection`, `runtimeMode`, `interactionMode`,
  `lastRunStatus` and `runCount` (`packages/contracts/src/scheduledTask.ts:234-257`, field at
  `:242`). The RPC upsert input also has `workspaceStrategy` (`:281`).
- The local route is `/api/hooks/<taskId>/<token>`
  (`apps/server/src/scheduledTasks/ScheduledTaskService.ts:70`, `:305-306`). The public route
  goes through the T3 Connect relay, `${relayUrl}/v1/hooks/${endpointKey}` (`:107`).
- Limits: up to 20 queued deliveries per task (`ScheduledTaskService.ts:80`) and a rate limit
  of 60 a minute (`:82`).
- The relay can hold webhooks for up to 24 hours while the environment is offline
  (`hold_webhooks_while_offline`). Each forward carries `x-t3-relay-delivery-id`
  (`docs/internals/t3-connect.md:16-24`).
- The prompt can use `{{body.path}}`, `{{headers.name}}` and similar placeholders. The values
  reach the agent inside `<webhook-data-…>` tags as untrusted data
  (`docs/user/project-settings.md:63-120`).
- Web: `apps/web/src/components/settings/ScheduledTasksSettings.tsx`. The default draft is a
  worktree from `main` with `startFromOrigin` (`:145`). A "Base branch" field exists
  (`:1420-1427`). The thread panel is `apps/web/src/components/chat/ThreadAutomationsPanel.tsx`.
- Mobile: `apps/mobile/src/features/settings/scheduled-task-editor.tsx` and
  `scheduledTaskDraft.ts` (base ref at `:195-276`, webhook modes at `:54-160`). The mobile app
  cannot turn on a signature (`docs/user/project-settings.md:63-120`).
- Web and mobile show the local webhook URL when there is no public URL. The shared helper
  builds it from the environment's HTTP base URL
  (`packages/client-runtime/src/webhookAddress.ts:10-33`).
- **Defect 1: the MCP path hard-codes the base branch.** `schedule_task` has no
  `workspaceStrategy` input (`packages/contracts/src/orchestratorMcp.ts:585-602`). The
  handler uses `{ type: "worktree", baseRef: "main", startFromOrigin: true }`, or `root` for a
  task bound to the thread (`apps/server/src/mcp/OrchestratorMcpService.ts:285-291`, used at
  `:1837` and `:1911`).
- **Defect 2: the MCP result hides the local hook path.** `webhookUrl` is "Absent when this
  environment has no T3 Connect managed tunnel"
  (`packages/contracts/src/orchestratorMcp.ts:615-618`). The record has the path
  (`packages/contracts/src/scheduledTask.ts:227-231`), but the MCP summary does not return it.

### 6.3 Threads, worktrees and delegation

- `delegate_task` starts a child thread on any provider instance and model
  (`apps/server/src/mcp/toolkits/orchestrator/tools.ts:68-79`). The input has `task`,
  `target` (`providerInstanceId`, `driverKind`, `model`, `options`), `title`, `role`,
  `isolation` (`shared` or `worktree`), `mode` (`async` or `wait`), `timeoutMs`,
  `runtimeMode` and `interactionMode` (`packages/contracts/src/orchestratorMcp.ts:96-119`,
  `:160-191`).
- The result has `taskId`, `childThreadId`, `workState`, `summary`, `hasPendingQuestion` and
  `pendingApproval` (`packages/contracts/src/orchestratorMcp.ts:194`).
- `role` is one of `implementation`, `research`, `review`, `design`, `test` or `general`
  (`packages/contracts/src/orchestratorMcp.ts:130-137`). **It only adds one line to the
  prompt:** "Act as the ${role} sub-agent for this task."
  (`apps/server/src/mcp/OrchestratorMcpService.ts:819-821`).
- Other tools: `task_status`, `task_cancel`, `create_threads`, `t3_thread_list`, `_read`,
  `_update` (rename, link a PR), `_send` (auto, queue, steer or restart), `_wait` and
  `_interrupt` (`apps/server/src/mcp/toolkits/orchestrator/tools.ts:81-264`).
- `t3_thread_launch` takes a workspace strategy: `worktree` (`baseRef`, `branch`,
  `startFromOrigin`), `existing_worktree` or `root`. A launched thread may not get broader
  modes than the caller (`apps/server/src/mcp/toolkits/project/tools.ts:104`).
- Worktree tools: `t3_worktree_handoff`, `_status`, `_list`
  (`apps/server/src/mcp/toolkits/worktree/tools.ts:28-61`).
- Thread tools: organize (pin, snooze, settle, archive, unread), the queue (list, read, edit,
  cancel, reorder, promote to steer), fork from a run or checkpoint, merge back, transfers,
  search and `run_scheduled_task_now` (`apps/server/src/mcp/toolkits/thread/tools.ts:31-257`).
- Threads record their lineage: `parentThreadId`, `relationshipToParent` (`fork` or
  `subagent`) and `rootThreadId` (`packages/contracts/src/orchestrationV2.ts:109-113`).
- A concurrency cap for threads or delegated tasks was not found. `DispatchModeLimit.ts`
  limits only the modes (`apps/server/src/orchestration-v2/DispatchModeLimit.ts`).

### 6.4 Plan mode, questions and approvals

- Plan mode is the `plan` interaction mode (`packages/contracts/src/providerPolicy.ts:25-36`).
  A proposed plan appears only where the adapter emits one (section 6.1).
- Structured questions are pending requests. The tools list, read and respond to them.
  "Approval requests are not included." (`apps/server/src/mcp/toolkits/thread/tools.ts:151-169`).
- A pending question holds a live turn. When the turn ends, the question is gone. T3 has no
  durable "waiting for a human decision" state with a reason. This is an inference from the
  read model, not a documented statement.

### 6.5 PRs and source control

- `link_pull_request` "settles the thread when it merges"
  (`apps/server/src/mcp/toolkits/pullRequests/tools.ts:273`).
- `watch_pull_request`: "T3 Code checks it every two minutes and wakes you with a message when
  a check fails, the required checks pass, someone else comments or reviews, or the branch
  starts to conflict with its base." (`apps/server/src/mcp/toolkits/pullRequests/tools.ts:317-320`).
  A subagent cannot watch. The sweep constant is `SWEEP_MINUTES = 2`
  (`apps/server/src/orchestration-v2/PullRequestWatchReactor.ts:43`).
- A watch ends after 10 comment-only wakes. A watched thread stays in Working
  (`docs/user/source-control.md:285-302`).
- Review guides (beta) give a tour, not a verdict. "Claude and Codex can write guides."
  (`docs/user/source-control.md:205-237`, `:223`; tools at
  `apps/server/src/mcp/toolkits/pullRequests/tools.ts:378-394`).
- PR actions: merge, ready, draft, close, reopen, delete branch, update branch, enable and
  disable auto-merge, revert, approve workflows (`packages/contracts/src/pullRequest.ts:63-75`).
- Auto-merge with outstanding checks works on GitHub, GitLab and Azure DevOps
  (`docs/user/source-control.md:183`).
- Source control drivers: GitHub, GitLab, Azure DevOps, Bitbucket, Forgejo and GitCafe
  (`apps/server/src/sourceControl/builtInDrivers.ts:23-30`). Forgejo is "Forgejo / Gitea" and
  uses `tea` 0.16+ or `fj` 0.6+
  (`packages/source-control-forgejo/src/server/ForgejoSourceControlProvider.ts:24`,
  `docs/user/source-control.md:37-56`).
- No issue-tracker or chat integration exists. Settings → Integrations has only Browser and
  Devices (`apps/web/src/components/settings/IntegrationsSettings.tsx:812`, `:1631`).

### 6.6 The human loop

- Working section: "A thread returns to the top" when it finishes, fails, or needs an
  approval or an answer (`docs/user/thread-sidebar.md:119-126`).
- Settle, snooze and pin exist. Auto-settle runs after 3 days or on a merged PR
  (`docs/user/thread-sidebar.md:131-160`).
- Mobile notifications fire "when an agent finishes, fails, needs approval, or asks for
  input". They need T3 Connect (`docs/user/mobile-notifications.md:3`).
- Mobile can edit, pause, run now or delete scheduled tasks, and review PRs with "Work on PR"
  (`docs/user/project-settings.md:63-120`, `docs/user/source-control.md:158-203`).
- The activity log shows orchestrator summaries (`docs/user/activity-log.md:8-10`).

### 6.7 Usage and run data

- The Usage page combines Codex, Claude Code, Grok Build, OpenCode, Antigravity, Pi and Cursor
  history. It shows tokens and estimated API-equivalent cost (`docs/user/usage.md:8-11`).
- A usage bucket has `day`, `hourStart`, `provider`, `model` and `sourcePath`. It has no
  project, thread, job or PR dimension (`packages/contracts/src/usage.ts:61-78`).
- Provider turns carry per-turn token usage (`packages/contracts/src/orchestrationV2.ts:167-178`).
  `TurnTokenUsage` has `usageStatus` `complete`, `partial` or `unavailable`
  (`packages/contracts/src/providerRuntime.ts:333-346`).
- Adapters that set `turnTokenUsage`: Claude, Codex, OpenCode (v1 and v2) and Muse, plus the
  shared `ProviderEventIngestor.ts`. For the other adapters this is not verified.
- `apps/server/src/orchestration-v2/RunMetrics.ts` exports OpenTelemetry run metrics only.

### 6.8 Remote, CLI and outside agents

- Outside agents connect to `/mcp` with an OAuth sign-in. Scopes go from "Read only" through
  "Supervised" to "Full access" (`docs/user/outside-agents.md:50-52`).
- CLI: `t3 thread start <project> <message>` with `--title`, `--provider`, `--model`,
  `--worktree <base-ref>`, `--branch`, `--wait` and `--json`
  (`apps/server/src/cli/thread.ts:535-612`, worktree at `:460-475`). It calls
  `t3_thread_launch`. `--wait` exits 1 on failure and 2 when the thread waits for an answer.
- The CLI has no flag for model options (such as effort) or for the runtime mode (same lines).
- Also `t3 thread send` (queue mode, `:614`), `show` (`:648`) and `list` (`:499`).
- `t3 triage` is T3's own support triage, not issue triage (`apps/server/src/cli/triage.ts:144`).
- No CLI command for scheduled tasks was found (`apps/server/src/cli/`).

### 6.9 Project file, knowledge and principles

- `t3.json` has `$schema`, `iconPath`, `defaultThreadEnvMode`, `worktreeSubmodules` and
  `scripts`. A script can run on worktree creation or on settle
  (`packages/contracts/src/t3ProjectFile.ts:38-53`). It defines no agents, roles or loops.
- Portable handoffs move budgeted context, not a summary (`docs/user/portable-handoffs.md`,
  `docs/internals/context-handoffs.md`).
- A memory feature was not found.
- Upstream's rule: "a capability is a service method". It is reachable from WS, MCP, scheduled
  tasks and the CLI (`docs/internals/effect-services.md:3-23`). Upstream `AGENTS.md` adds: "A
  capability a user can trigger is usually one an agent should reach through MCP tools, and
  scheduled tasks run the same paths."

## 7. The user's factory (factory-hub)

### 7.1 What it does

- It is a Claude Code plugin served from a Gitea marketplace and run as `/factory:run`
  (`factory-hub/docs/decisions.md:10`).
- **The loop** (`factory-hub/plugin/skills/run/SKILL.md:56-127`): claim a job, make a
  worktree, write a spec, review the spec, ask the owner, build, review the code, rework, open
  the PR, wait for CI, do a docs pass and QA, approve, merge queue, sync and end.
- **Roles:** 11 agents (`factory-hub/plugin/agents/*.md`): spec-writer, builder, approver,
  qa, review-axis, review-ba, review-code, review-legal, review-security, review-ui and
  review-ux. A roster decides which reviewers join (`SKILL.md:129-183`).
- **Config layers:** `plugin/core.toml` → packs → `.factory/config.toml` → role addenda.
  `factory brief` prints the result (`factory-hub/docs/decisions.md:11`). Packs:
  `legal-eu-ro`, `nextjs-drizzle-trpc`, `web-ui` (`factory-hub/plugin/packs/`).
- **Limits:** `activeJobs = 3`, `prSlots = 0`, `freeReworks = 2`, `maxReworks = 4`,
  `maxSpecRounds = 5` (`factory-hub/plugin/core.toml:19-25`).
- **Models:** every agent runs on Opus. The effort comes per role from `models.effort`
  (`factory-hub/plugin/core.toml:42-53`, decision 31 at `factory-hub/docs/decisions.md:40`).
- **Gates:** questions go through triage first; only owner decisions reach the user
  (`SKILL.md:272-291`). The AI approver writes `decision.md`. Anything but `APPROVE` goes to
  `needs-human` with a reason (`SKILL.md:496-505`).
- **needs-human is durable.** The job keeps its stage and reason in `board.json`, releases
  its locks and ends. `/factory:run approve <job>` or `rework <job> <note>` resumes it later
  (`SKILL.md:608-645`).
- **The factory merges by itself** after `APPROVE`. It takes a merge lock and relies on the
  forge's branch protection (`SKILL.md:535-545`). This differs from Warp (W5).
- **Engine:** the `factory` CLI (Python) keeps `board.json` under flock, locks, PR slots, a
  merge lock, CI polling and metrics (`factory-hub/plugin/lib/engine/board.py:1-48`).
- **Gitea:** `tea api` for blockers and commit statuses (`factory-hub/plugin/lib/engine/gitea.py:1-3`, `:33`, `:41`).
- **Metrics:** per job, the tokens and active time of each agent type, claim-to-merge time,
  time per stage, review rounds, reworks, escalations, needs-human stops, and escaped-defect
  candidates (`factory-hub/plugin/lib/engine/metrics.py:1-33`).
- **The hub** (Python, port 3150) shows an inbox across projects, boards, runs and a dashboard
  (`factory-hub/hub/server.py:10-21`). The dashboard has autonomy, cycle hours, cost per
  role, needs-human and escalations (`factory-hub/hub/model.py:173-216`).
- **Hub tier 3** starts, approves and reworks jobs through three T3 webhook tasks per project:
  `/factory:run next`, `approve {{body.job}}` and `rework {{body.job}} {{body.note}}`
  (`factory-hub/docs/hub.md:107-121`).

### 7.2 Provider-agnostic parts

These parts do not depend on Claude Code. They are candidates to keep, or to express with T3
building blocks:

| Part | Where |
| ---- | ----- |
| The loop and its stages | `plugin/skills/run/SKILL.md:56-127` |
| The verdict contract (line 1 is the verdict) | `plugin/skills/run/SKILL.md:381-396`, `plugin/agents/*.md` |
| Config layers, packs and `factory brief` | `plugin/core.toml`, `plugin/packs/`, `docs/decisions.md:11` |
| Roster rules and convergence caps | `plugin/skills/run/SKILL.md:129-183`, `plugin/core.toml:19-25` |
| The approver's escalation rules | `docs/decisions.md:17`, ADR 0005 |
| Follow-up and counsel issues through `tea` | `plugin/skills/run/SKILL.md:293-332`, `:507-533` |
| The engine: board, locks, PR slots, merge queue, CI polling | `plugin/lib/engine/board.py`, `plugin/skills/run/SKILL.md:535-564` |
| Escaped-defect detection with `git blame` | `plugin/lib/engine/metrics.py:28-33` |
| The hub UI and its actions | `hub/server.py`, `hub/actions.py` |

### 7.3 Claude-specific parts

| Part | Where | T3 building block that could replace it |
| ---- | ----- | --------------------------------------- |
| Plugin packaging, the marketplace and the `factory:` namespace | `docs/decisions.md:10`, ADR 0001 | None needed. The loop can stay a Markdown playbook in the repo. |
| Spawning roles with Claude's Agent tool (`subagent_type`, `model: "opus"`, `effort`) | `plugin/skills/run/SKILL.md:151-164` | `delegate_task` with `target.options` and the role file as the task (section 6.3). |
| Resuming a cut-off agent with `SendMessage` | `plugin/skills/run/SKILL.md:381-396` | `t3_thread_send` to the `childThreadId`. |
| Agent frontmatter: `tools`, `disallowedTools`, `skills` (mattpocock-skills, ui-ux-pro-max) | `plugin/agents/*.md` | No portable equivalent. Tool lists and preloaded skills differ per provider. |
| The effort semantics (`xhigh`, `high`, `medium`) | `plugin/core.toml:42-53` | `target.options` per provider. The values differ per provider. |
| Grilling through the Claude skill `mattpocock-skills:grilling`, and answers through AskUserQuestion | `plugin/skills/run/SKILL.md:283-288`, `plugin/lib/engine/metrics.py:13-14` | Structured questions, where the adapter supports them (not on Cursor). |
| Locks keyed on `CLAUDE_CODE_SESSION_ID` and the nearest `claude` process | `plugin/lib/engine/board.py:144-152`, `:247` | The T3 thread id. `FACTORY_OWNER_PID` already overrides the process walk (`board.py:146-147`). A T3 environment variable with the thread id was not found. |
| Metrics from `~/.claude/projects` transcripts | `plugin/lib/engine/metrics.py:1-2`, `:78-81` | Per-turn token usage in T3, if T3 exposes a per-thread roll-up (gap G5). |

### 7.4 Where it touches T3 today

- The hub reads T3's live state database read-only (`mode=ro`), table `scheduled_tasks`, to
  find the webhook tokens (`factory-hub/hub/hubctl.py:61-66`, `:104`). This couples the hub to
  T3's internal schema. Defect 2 (section 6.2) causes it.
- The hub moves the project's local `main` with `git branch -f main <base>` before each
  request (`factory-hub/hub/actions.py:187`, `:249`). This is the work-around of decision 24
  (`factory-hub/docs/decisions.md:33`). Defect 1 (section 6.2) causes it. Decision 11 asks T3
  for a base-branch field (`factory-hub/docs/decisions.md:20`).
- Open items in `factory-hub/docs/HANDOFF.md`: one usage limit across projects (`:69-73`),
  decision 11 not filed (`:74-77`), no HMAC on the hub's T3 tasks while T3 binds
  `0.0.0.0:3773` (`:81-84`).
- The run skill calls `link_pull_request` "when that tool exists" (`SKILL.md:436`).
- `factory-hub/docs/improvement.md:207-216` lists "Not adopted": a factory.yaml, model routing
  and other harnesses, Factory MCP and Slack.

## 8. Gap matrix

Legend: **Yes** = present. **Part** = partial. **No** = absent or not found. "Prev" = Private
Preview or Early Access. The evidence is in sections 4-7 under the IDs given.

| # | Capability | Factory | Warp | T3 today | User's factory |
| - | ---------- | ------- | ---- | -------- | -------------- |
| C1 | Factory definition as code (agents and loop in the repo) | Part: droids, skills, hooks as files (F3, F4) | Yes: `factory.yaml` tree (W1) | No: `t3.json` has scripts only (6.9) | Yes: plugin, packs, config layers (7.1) |
| C2 | Role agents with own prompt and model | Yes (F3) | Yes (W2) | Part: `delegate_task` target and options; `role` is one prompt line (6.3) | Yes: 11 agents, Opus, effort per role |
| C3 | Orchestrator runs roles through stages | Yes: Missions (F5) | Yes: foreman (W3) | Part: tools only; the orchestrator is an agent (6.3) | Yes: run skill (7.1) |
| C4 | Spec or plan mode with approval | Yes: Spec Mode (F6) | Yes: spec gate (W4) | Part: `plan` mode, per adapter (6.4) | Yes: spec, BA review, questions |
| C5 | Schedule triggers | Yes (F8) | Yes (W6) | Yes (6.2) | Via T3 |
| C6 | Webhook triggers with signatures | Prev (F8) | Yes (W6) | Yes; MCP lacks base branch and local path (6.2) | Via T3, no HMAC (7.4) |
| C7 | Issue-tracker triggers | Yes: Linear, Jira (F9) | Yes: GitHub, GitLab, Azure DevOps, Linear, Jira (W6) | No native; generic webhook (6.2) | No: a queue in `backlog.md`, the hub starts jobs |
| C8 | Chat triggers | Yes: Slack, Teams (F9) | Yes: Slack, Teams (W6) | No (6.5) | No; "not adopted" |
| C9 | Isolated worktree per job | Yes (F12) | Sandbox per run (W8) | Yes (6.3) | Yes, via its own worktrees |
| C10 | Hosted compute | Yes (F13) | Yes; required (W8) | No; your own machines (6.8) | No; VM `agents` |
| C11 | Headless run: CLI, API, SDK | Yes (F1, F2) | API and SDK; no CLI (W7) | CLI and WS RPC (6.8) | Via T3 webhooks |
| C12 | MCP for outside agents | Not found | Yes, no read-only scope (W7) | Yes, with scopes (6.8) | No |
| C13 | Automated PR review | Yes (F11) | Advisory review agent (W2) | Part: review guides, a tour (6.5) | Yes: reviewer roster and approver |
| C14 | PR babysitting (CI, comments, conflicts) | Yes: PR Babysitter (F10) | Part: reviewing stage; CI fixes not verified | Yes: `watch_pull_request` (6.5) | Yes: CI polling and rework |
| C15 | Merge control | Not found | Repo-enforced, never merges (W5) | Part: PR actions, auto-merge on 3 hosts (6.5) | Yes: merge lock, forge protection |
| C16 | Human inbox with typed waits | Part: Mission Control, events (F15) | Yes: 6 item types (W10) | Part: Working section, live questions (6.6) | Yes: hub inbox, durable needs-human |
| C17 | Mobile and push | Browser only (F19) | Not found | Yes: app and push (6.6) | Via T3 |
| C18 | Autonomy and permission levels | Yes (F7) | Not found for factories | Yes: runtime modes, per adapter (6.1) | Full access via T3 |
| C19 | Per-job measures: cost, cycle time, autonomy | Part: Analytics, Effectiveness (F15) | Yes (W11) | No: usage by provider, model, day (6.7) | Yes: metrics and dashboard |
| C20 | Scorers and self-improvement | Part: "Memory & Evolution" (F17) | Yes (W12) | No | Part: Score step; retro not built |
| C21 | Many models, router, BYOK | Yes (F14) | Yes: 3 harnesses, routers (W9) | Yes: 9 drivers; no router (6.1) | No: Opus only, by choice |
| C22 | Org knowledge | Part: AGENTS.md, AutoWiki (F17) | Part: skills; memory marketing only (W13) | Part: AGENTS.md through providers (6.9) | Part: docs pass, project docs |
| C23 | Secret scan before push | Yes (F18) | Not found | No | No |
| C24 | Gitea support | No for cloud features (F20) | No (W6) | Yes: Forgejo/Gitea driver (6.5) | Yes: `tea` |
| C25 | Multi-repo and shared config | Part: org plugin marketplaces | Yes: repos per factory, no inheritance (W15) | No | Yes: packs and core |
| C26 | Concurrency cap | Not found | Not found | Not found (6.3) | Per project only (7.4) |
| C27 | Access roles and scopes | Yes: SSO, SCIM (F21) | Yes: 3 roles (W14) | Part: MCP scopes, pairing scopes (6.8) | Hub token (ADR 0006) |

The gaps where T3 is "No" or "Part", and that the user's factory needs, are:

| Gap | Capabilities | Who fills it today |
| --- | ------------ | ------------------ |
| G1: MCP base branch and local hook path | C6 | The hub: `git branch -f main` and a read of T3's database |
| G2: committed role definitions | C1, C2 | The plugin's agent files (Claude-only frontmatter) |
| G3: durable gate with a reason | C16 | `board.json` stage `needs-human` and the hub inbox |
| G4: tracker triggers | C7 | The hub and `backlog.md` |
| G5: per-job measures | C19 | `metrics.py` over Claude transcripts |
| G6: concurrency cap across projects | C26 | Nobody (`HANDOFF.md:69-73`) |

## 9. Recommendation

This section is opinion. Sections 4-8 are the facts.

### 9.1 Constraints from AGENTS.md

- One decision per adapter for every provider-shaped feature.
- Web, desktop and mobile. Local, LAN or Tailscale, and T3 Connect.
- No extra data on the WebSocket. Compute measures on demand.
- Wire types live in `packages/contracts`.
- "Orchestration stays pure, UI stays dumb." Complexity goes to the adapter boundary.
- Theo's note: "Do not introduce machinery because it looks architecturally impressive."
- Upstream's rule: a capability is a service method that WS, MCP, scheduled tasks and the CLI
  reach (`docs/internals/effect-services.md:3-23`).

### 9.2 How the loop maps onto T3

```
trigger: hub button, Gitea webhook, schedule        [exists: webhook task; G1 base branch]
   │
   ▼
orchestrator thread (any adapter with T3 MCP tools, worktree from origin/<base>)
   │  delegate_task(target, options, role file as task)          [exists; G2 presets later]
   ├──► spec-writer child ──► spec reviewers (children, parallel)
   ├──► builder child (worktree)
   │  link_pull_request + watch_pull_request                      [exists]
   ▼
needs-human ──► gate in inbox and push ──► reply: approve or rework       [G3]
   │
   ▼
merge: forge branch protection + engine merge lock                [exists outside T3]
   │
   ▼
measures: roll-up of the root thread and its children            [G5]
```

### 9.3 Steps in order

**Step 0. Base the fork on upstream (at least `v0.1.3-nightly.20261023.3566`).** Done on
2026-10-26: a fast-forward to upstream `a240408242`, with no fork-only commits to keep.
- What changes: the fork gets scheduled tasks, the orchestrator MCP tools, PR watch and the
  Forgejo/Gitea driver. The running T3 already has them.
- Cost: a merge of about two months of upstream.
- What can break: fork-only changes, if any. There were none.
- The user chose to keep the features in the fork, not to contribute them upstream
  (open question 1).

**Step 1. Port the run skill to T3 MCP tools. No T3 change.**
- What changes: the Agent tool becomes `delegate_task` with `target.options` and the role
  file as the task text. `SendMessage` becomes `t3_thread_send` to the child thread.
  AskUserQuestion becomes a structured question. Locks key on the T3 thread id.
- Cost: work in factory-hub only. The orchestrator must know provider instance ids, from
  `orchestrator_capabilities` (`apps/server/src/mcp/toolkits/orchestrator/tools.ts:54`).
- What can break: tool lists and preloaded skills in agent frontmatter are lost. Each role
  file must then state its limits in text. Grilling does not work on Cursor (no structured
  questions). The orchestrator cannot run on OpenCode's v2 runtime (section 6.1).
- Per adapter: decide per role which providers are allowed. The user's rule today is Opus
  only (open question 3).
- Why first: it proves which T3 gaps are real before T3 gets new features.

**Step 2. Fix G1 in the MCP layer.**
- What changes:
  - `OrchestratorMcpScheduleTaskInput` and the update input get an optional
    `workspaceStrategy` (`packages/contracts/src/orchestratorMcp.ts:585-602`). It reuses
    `OrchestrationV2ThreadLaunchWorkspaceStrategy`, as `ScheduledTask` does
    (`packages/contracts/src/scheduledTask.ts:242`).
  - The handler keeps today's default when the field is absent
    (`apps/server/src/mcp/OrchestratorMcpService.ts:285-291`).
  - `OrchestratorMcpScheduledTask` returns the local `webhookPath` next to `webhookUrl`
    (`packages/contracts/src/orchestratorMcp.ts:615-618`).
- Cost: small. One contract, one handler and focused tests. Web and mobile need no change:
  they already edit the base branch and show the local URL.
- What can break: an agent can then start tasks from any ref. The rule "no broader modes than
  the caller" should also apply here (not verified for scheduled tasks). The path contains the
  token. Anyone who reads the tool result can start runs on the LAN, because T3 binds
  `0.0.0.0:3773` (`factory-hub/docs/HANDOFF.md:81-84`). A signature reduces this risk.
- Per adapter: none. The server launches the thread; adapters are not touched.
- Connection modes: the path is relative. It works locally, on the LAN and over Tailscale. A
  public URL still needs T3 Connect.
- Effect on the user's factory: the hub drops `git branch -f main` and its read of T3's
  database. Decision 11 is met.
- Alternative with no T3 change: the hub calls `t3 thread start <project> "/factory:run next"
  --worktree origin/<base> --provider … --model …`. Cost: no effort flag and no runtime-mode
  flag (section 6.8). The hub also loses the webhook queue and the run records.

**Step 3 (only if step 1 shows the need). Committed agent presets (G2).**
- What changes: a project can commit named presets. Each preset has instructions (a Markdown
  file), a target (provider instance, model, options), an isolation and a runtime mode.
  `delegate_task` accepts `agent: "<name>"`. The presets could live under a key in `t3.json`
  (`packages/contracts/src/t3ProjectFile.ts`) or as `agents/*.md` files. Open question 2.
- Cost: medium. A contract, a resolver in the MCP service, a picker in web and mobile, and docs
  in `docs/user/`.
- What can break: a preset names a provider instance that a remote environment lacks. The
  resolver must fail with a clear message, not fall back silently.
- Per adapter: instructions, model, options and runtime mode are portable. Tool allow-lists
  and preloaded skills are not portable; mark them "not supported" in the first version.
  Cursor has no command approval, so `approval-required` means less there.
- Why only conditional: step 1 may show that the role file as task text is enough.

**Step 4. A durable gate on a thread (G3).**
- Zero-change option first: the agent ends its turn with a clear last message. T3 already
  moves a finished thread to the top and sends "finished" to mobile (section 6.6). Cost:
  nothing. Loss: no reason label, no filter, no "needs decision" push.
- What changes in the full option: an agent sets a gate with a reason and optional choices
  through `t3_thread_update`. The decider records it as an event. The projector shows it in
  the inbox, the Working section and mobile, and push says "needs a decision". The next user
  message clears it. A "clear gate" action is the way out, also for a gate that nobody needs.
- Cost: medium-small. A contract, one command and one event, a badge in web and mobile, a
  push category, docs in `docs/user/thread-sidebar.md`.
- What can break: an agent can forget to set a gate, or set one without need. T3 does not
  enforce the stage; the gate is a label that the agent sets. Old clients ignore the new
  field (not verified).
- Per adapter: none. Every adapter with T3 MCP tools can set it. Generic ACP agents use the
  `acp-mcp-call` fallback.
- Effect on the user's factory: `needs-human` shows in T3 on every surface. The hub inbox
  becomes optional (open question 5).

**Step 5. A per-thread measure read (G5).**
- What changes: one read-only MCP tool and one RPC method. They sum per-turn token usage and
  turn time over a root thread and its children (lineage at
  `packages/contracts/src/orchestrationV2.ts:109-113`). They also return the number of user
  messages and the linked PR's open and merge times.
- Cost: small to medium. Computed on demand from the read model. Nothing new on the WebSocket
  stream. No new storage.
- What can break: partial data looks complete. The result must count turns by `usageStatus`
  (`complete`, `partial`, `unavailable`).
- Per adapter: Claude, Codex, OpenCode and Muse set turn usage. For Cursor, Grok, Antigravity
  and Pi this is not verified. Report them as "unavailable" until checked.
- Effect on the user's factory: `metrics.py` no longer reads Claude transcripts. Measures work
  for every provider.

**Later, only with evidence:**
- A Jobs view across projects (autonomy, cycle time, cost per PR). This is the hub's
  dashboard. Build it in T3 only if the hub is to go away.
- Signature presets for Gitea and GitHub webhook senders. The generic signature check already
  covers both (section 6.2).
- A concurrency cap across projects (G6). The engine can enforce it with `t3_thread_list`
  first.

### 9.4 Not recommended now

| Idea | Reason |
| ---- | ------ |
| A workflow language or stage engine in orchestration | The loop changes often and is project policy. Warp also keeps gates as "workflow policy, written into the foreman's instructions" (W4). A playbook in the repo keeps orchestration pure. |
| A Factory Droid provider | The user rejected it. |
| Native Slack, Linear or Jira integrations | Not used by the user (`factory-hub/docs/improvement.md:215`). A generic webhook covers Gitea. |
| Hosted compute | T3's model is the user's own environments. |
| An org memory store | Neither Factory nor Warp documents one as a factory feature (F17, W13). |
| An automatic model router | The user runs Opus only, by choice (`factory-hub/docs/improvement.md:211-214`). |
| A merge queue in T3 | Forge branch protection plus the engine's merge lock work. T3 has no auto-merge on Gitea (section 6.5). |
| Agent scorers in T3 | The Score step and the retro are open work in factory-hub (`factory-hub/docs/improvement.md:139-193`). Measure first (step 5). |

## 10. Open questions for the user

1. Do you want to contribute these features upstream (pingdotgg/t3code), or keep them in your
   fork? Upstream review changes the scope and the order. **Answer (2026-10-26): keep them in
   the fork.**
2. Where should committed presets live: a key in `t3.json`, `agents/*.md`, or nowhere (the
   loop passes the role file as text)?
3. Which providers may each role use? Does the Opus-only rule still hold after the port?
   **Answer (2026-10-26): Claude only, for now.**
4. Do you want a concurrency cap across projects? Should T3 or the engine own it?
5. Should the hub stay, or should T3's inbox and a later Jobs view replace it?
6. Will you link T3 Connect for public webhooks and mobile push, or stay on the LAN?
7. Should Gitea issue events (for example a label) start jobs, through a signed webhook task?
8. Is a gate inside T3 wanted, or are the hub's approve and rework buttons enough?
9. Who owns the Score and retro work: the factory or T3?
10. May I file decision 11 (the base-branch request) as an upstream issue? Filing posts to an
    outside service (`factory-hub/docs/HANDOFF.md:74-77`).

## 11. Not verified

- Whether Droid can run through T3's generic ACP driver.
- A real Gitea webhook delivery to a T3 webhook task with `X-Gitea-Signature`.
- Whether the "no broader modes than the caller" rule applies to MCP-created scheduled tasks.
- Which adapters other than Claude, Codex, OpenCode and Muse set per-turn token usage.
- Whether old clients ignore a new thread field without errors.
- A T3 environment variable that gives an agent its own thread id.
- Factory: a "shared" level in the droid precedence rule; a native mobile app; a Linux desktop
  build; an org memory feature; a page that says "only GitHub and GitLab"; `/handoff` and
  `/migrate` in the docs (changelog only); the plan of the audit log (sources conflict); an
  MCP server for outside agents.
- Warp: Cursor or OpenCode as factory harnesses (marketing only); memory as a factory feature;
  skill discovery in referenced repos; the agent-hour rates; the GA date; CI-fix behavior in
  the reviewing stage; a mobile app.

## 12. Sources

All web sources were read on 2026-10-26.

Factory (docs pages read as `.md` from the index <https://docs.factory.com/llms.txt>):

- <https://docs.factory.com/droid-exec/overview>, <https://docs.factory.com/droid-cli/overview>,
  <https://docs.factory.com/droid-cli/settings>, <https://docs.factory.com/sdk/python>,
  <https://docs.factory.com/api-reference/sessions>
- <https://docs.factory.com/harness/subagents>, <https://docs.factory.com/harness/plugins>,
  <https://docs.factory.com/harness/hooks>, <https://docs.factory.com/harness/skills>
- <https://docs.factory.com/missions/overview>, <https://docs.factory.com/missions/planning>
- <https://docs.factory.com/autonomy-and-safety/specification-mode>,
  <https://docs.factory.com/autonomy-and-safety/auto-run>,
  <https://docs.factory.com/autonomy-and-safety/permission-rules>,
  <https://docs.factory.com/autonomy-and-safety/droid-shield>
- <https://docs.factory.com/software-factory/overview>,
  <https://docs.factory.com/software-factory/automations>,
  <https://docs.factory.com/software-factory/code-review-ci>,
  <https://docs.factory.com/software-factory/release>,
  <https://docs.factory.com/software-factory/droid-control>,
  <https://docs.factory.com/software-factory/wiki/auto-refresh>
- <https://docs.factory.com/delegations>, <https://docs.factory.com/factory-app/worktrees>,
  <https://docs.factory.com/droid-computers/overview>,
  <https://docs.factory.com/environments/overview>,
  <https://docs.factory.com/opencompute/overview>, <https://docs.factory.com/remote-control>
- <https://docs.factory.com/model-independence/factory-router>,
  <https://docs.factory.com/model-independence/byok>, <https://docs.factory.com/models>
- <https://docs.factory.com/agent-effectiveness/cost-and-productivity>,
  <https://docs.factory.com/agent-readiness/overview>
- <https://docs.factory.com/enterprise/self-managed-scm>,
  <https://docs.factory.com/enterprise/identity-and-access>,
  <https://docs.factory.com/enterprise/audit-log>,
  <https://docs.factory.com/enterprise/internal-plugin-marketplaces>
- <https://docs.factory.com/pricing/individuals>,
  <https://docs.factory.com/changelog/release-notes>,
  <https://docs.factory.com/changelog/feature-maturity>
- Marketing: <https://factory.com/>, <https://factory.com/pricing>,
  <https://factory.com/news/automations>,
  <https://factory.com/news/factory-analytics-transparency>,
  <https://factory.com/news/factory-private>

Warp:

- <https://docs.warp.dev/factories/>, <https://docs.warp.dev/factories/how-factories-work/>,
  <https://docs.warp.dev/factories/factory-as-code/>,
  <https://docs.warp.dev/factories/factory-agents/>,
  <https://docs.warp.dev/factories/factory-agents/configuration/>,
  <https://docs.warp.dev/factories/connect-your-factory/>,
  <https://docs.warp.dev/factories/factory-api/>, <https://docs.warp.dev/factories/factory-mcp/>,
  <https://docs.warp.dev/factories/infrastructure-and-security/>,
  <https://docs.warp.dev/factories/runners/>,
  <https://docs.warp.dev/factories/factory-inbox/>,
  <https://docs.warp.dev/factories/factory-dashboard/>
- <https://docs.warp.dev/agents/inference/custom-routers/>,
  <https://docs.warp.dev/agents/agent-memory/>, <https://docs.warp.dev/platform/overview/>,
  <https://docs.warp.dev/platform/transitioning-from-oz/>,
  <https://docs.warp.dev/_llms-txt/changelog.txt>
- Blog: <https://www.warp.dev/blog/open-infrastructure-for-building-a-software-factory>
  (2026-08-18) and "The factory stack" (2026-09-05).
- Marketing: <https://www.warp.dev/pricing>
- Examples: <https://github.com/warpdotdev/warp-factory-examples>

T3 Code: upstream tag `v0.1.3-nightly.20261023.3566` (commit `807319c276c6`), paths as cited.

factory-hub: `/home/admin/factory-hub` at the state of 2026-10-26, paths as cited.
