---
name: syncast-cli
description: Install and use Syncast CLI for Syncast project workflows or standalone media generation. Before any generation, determine whether the user is working in a Syncast project or only wants independent assets for external viewing; project work must connect through project-agent and generate through project Actions.
---

# Syncast CLI

Syncast CLI lets external agents operate an opened Syncast project or call
standalone image, video, audio, music, and sound-effect generation APIs from the
terminal.

## Mandatory routing before any generation

Before running an image, video, audio, music, sound-effect, upscale, or other media
generation command, determine where the work belongs.

- If the user already refers to a Syncast project, its Docs, Assets, channels,
  canvas, timeline, or existing project media, treat the task as **project
  work** without asking again.
- If the destination is unclear, stop and ask: **"Is this work for a Syncast
  project, or do you only want independent assets to review outside a
  project?"** Do not generate until the user answers.
- If the user chooses project work, install/read `syncast-agent-actions`, start
  or reuse `syncast project-agent`, connect the intended project page, run
  `syncast.project.inspect`, and perform every generation through the project's
  public Actions. Do not use `syncast imagine`, `syncast video`, `syncast
  audio`, `syncast music`, or `syncast sound-effect` as a fallback for project
  work, including with `--project`.
- Use the direct generation commands only after the user explicitly chooses
  independent assets for external viewing, or explicitly requests the
  standalone generation API.

This routing decision is about whether the user is working in a project, not
whether they explicitly requested ImagineChannel history. Project work must
leave its prompts, parameters, task state, results, and assets in the project.

## Keep external operations out of project content

This skill is an external integration guide, so it may describe CLI commands,
the Bridge, public Actions, and GraphQL. Before writing any project Skill,
Agent instructions, project spec, prompt template, or user document, classify
the target audience:

- External Agent integration guides may contain these implementation details.
- Project-internal Skills/specs/templates contain only business rules,
  project-capability semantics, model ids, and creative parameters. Never copy
  CLI, Bridge, standalone API, `syncast.*` Action names, GraphQL mutations,
  transport, or access instructions into them unless the user explicitly asks
  for integration documentation.
- User-facing project documents use product and creative language and do not
  expose access implementation, except for an explicitly requested developer
  or integration document.

Validate every text-bearing field, including name/title, description,
instructions/body, template fields, and metadata. For example, an internal
Skill should say “Use the project's Imagine capability with
`oai-gpt-image-2`, 16:9, 2K,” not “call `syncast.imagine.submit` and do not use
the standalone CLI.”

## Prerequisites

- Node.js 18+ (npm/npx)
- A Syncast account

## Step 1: Install

```shell
npm install -g syncast-cli
```

Optional: install this skill for Cursor / compatible agents:

```shell
npx skills add latentcat/syncast-skills --skill syncast-cli -y
```

Install or refresh the project Agent Action skill when using `syncast project-agent`:

```shell
npx skills add latentcat/syncast-skills --skill syncast-agent-actions -y
```

CLI releases are triggered by Git tags named `syncast-cli-v<version>` where
`<version>` must match `node_packages/syncast-cli/package.json`.

## Skill updates

The Syncast CLI npm package and the Agent skills are updated separately. Installing or updating `syncast-cli` does not update the local `syncast-cli` or `syncast-agent-actions` skill files.

Before operating a project through `syncast project-agent`, and whenever an action/schema/example seems missing or stale, treat `latentcat/syncast-skills` as the source of truth and refresh the relevant skill:

```shell
npx skills add latentcat/syncast-skills --skill syncast-cli -y
npx skills add latentcat/syncast-skills --skill syncast-agent-actions -y
```

## Step 2: Log in

Run device authorization (opens browser or shows a link + code):

```shell
syncast auth login
```

Switch environment (dev is default):

```shell
syncast auth login --env nightly
syncast auth login --env local
```

For a fully custom API endpoint:

```shell
syncast auth login --api-url https://your-api.example.com
```

## Step 3: Verify

```shell
syncast auth status
```

Expected: `logged_in: true` with your username.

## Step 4: Generate standalone assets

The commands in this section are only for the independent-asset route selected
above. They are not the project workflow.

### Image

```shell
syncast imagine --prompt "a cat on the moon" --model nano-banana-2 --aspect-ratio 16:9
```

Use a local image file as a reference/input image:

```shell
syncast imagine \
  --prompt "turn this into a studio product photo" \
  --model nano-banana-2 \
  --reference-image ./reference.png
```

For multiple local reference images, repeat `--reference-image`:

```shell
syncast imagine \
  --prompt "combine the character from the first image with the outfit style from the second" \
  --model nano-banana-2 \
  --reference-image ./character.png \
  --reference-image ./outfit.png
```

Pass advanced image schema fields directly with `--input` or `--input-file`:

```shell
syncast imagine \
  --prompt "passport photo" \
  --model seedream-5-0 \
  --resolution custom \
  --width 2400 \
  --height 3600
```

Call OpenAI's official GPT Image 2 route explicitly when the user requests it:

```shell
syncast imagine \
  --prompt "precise bilingual launch poster" \
  --model oai-gpt-image-2 \
  --aspect-ratio 16:9 \
  --resolution 2K \
  --quality auto
```

Do not obtain or pass project storage keys. Standalone generation uses
`--reference-image` for local files. If an input is already a project Asset,
the task is project work: connect the project and pass its stable Asset ID to
the relevant project Action.

### Video

```shell
syncast video --prompt "a horse running on the beach" --duration 10
```

### Complete audio / scene-aware dubbing

Use SeedAudio for dubbing, dialogue, narration, audio drama, or a complete sound scene that mixes voices with ambience, SFX, or BGM:

```shell
syncast audio --prompt "一段约 60 秒的中文悬疑广播剧：两人低声对话，雨声和远处车流铺底，结尾出现急促脚步和刹车声"
```

### Music

Use `syncast music` for songs, instrumentals, BGM, scores, lyrics, extensions, and covers. It defaults to `zhenzhen-suno-v5.5`:

```shell
syncast music --prompt "warm cinematic score, slow piano and strings, restrained verse building into an expansive chorus"
syncast music --prompt "retro synthwave chase theme, 118 BPM" --instrumental
syncast music --mode extend --continue-at 28 --reference-audio ./source.mp3 --prompt "continue in the same key and build into a final chorus"
```

### Sound effect

```shell
syncast sound-effect \
  --prompt "seamless quiet office ambient sound, soft air conditioning, subtle computer hum, occasional keyboard typing, no voices, no music" \
  --duration-seconds 8 \
  --loop
```

### Human-readable output

```shell
syncast imagine --prompt "sunset over mountains" --format human
```

Default output is JSON (for agents).

### Generate inside a Syncast project

Project work always uses the opened-page Action Bridge. Connect and inspect the
project before generating:

```shell
syncast project-agent serve
syncast project-agent pages
syncast project-agent run syncast.project.inspect --input '{"limit":10}'
syncast project-agent capabilities --action syncast.imagine.submit --disclosure full
```

When the user did not specify an existing Imagine channel, use
`syncast.imagine.submit`. It automatically resolves or creates the Agent
Imagine channel while using the frontend enqueue and persistence path:

```shell
syncast project-agent run syncast.imagine.submit \
  --input '{"modelType":"nano-banana-2","prompt":"Generate a cinematic character sheet.","targetAssetName":"角色A首帧"}'
```

When the user explicitly selected an existing Imagine channel, use its real
`channelId` with `syncast.imagine.submitToChannel`:

```shell
syncast project-agent capabilities --action syncast.imagine.submitToChannel --disclosure full
syncast project-agent run syncast.imagine.submitToChannel \
  --input '{"channelId":"<imagine-channel-id>","modelType":"nano-banana-2","prompt":"Generate a cinematic character sheet.","targetAssetName":"角色A首帧"}'
```

Both project routes retain the prompt, parameters, task state, result messages,
and generated Assets in the project. `submit` is the default project generation
route; `submitToChannel` is only for a known existing channel.

The direct command's `--project`, `--name`, `--folder`, `--folder-id`, and
`--reference-asset` options remain low-level compatibility features for API
integrations. They do not reproduce project interaction and must not be chosen
by an external Agent after the user says the task is project work.

Document Imagine blocks are a third project-local contract. They keep an
editable root plus independent frozen generation versions, so they must use the
Action Bridge:

```shell
syncast project-agent capabilities --action syncast.docs.imagineBlocks.history --disclosure full
syncast project-agent run syncast.docs.imagineBlocks.submit \
  --input '{"docId":"<doc-id>","blockId":"<block-id>"}'
```

Every explicit `submit` creates another version, including after results exist
or while older versions run. `submitBatch` without `blockIds` is “generate all
pending”; explicit `blockIds` create another version for those blocks in
parallel. Wait on each returned `ref`, then call
`syncast.docs.imagineBlocks.history`. Copy one exact
`versions[].results[]` object into `selectResult` to change the preview or
`fixAsAsset` to land it as a normal Asset block. Do not fix while
`activeVersionCount > 0`. Use `restoreGeneration` to resume editing/generation
without losing history. Direct `syncast imagine`, even with `--project`, and
manual GraphQL state patches cannot replace this contract.

## Task management

```shell
# Stream / wait for an existing task
syncast task status <task_id>

# Cancel
syncast task cancel <task_id>
```

## Project Agent bridge

Use `syncast project-agent` when an external Agent needs to operate an opened
Syncast project through the narrow Agent Action bridge instead of injecting
arbitrary browser JavaScript.

```shell
syncast project-agent serve
syncast project-agent pages
syncast project-agent capabilities
syncast project-agent capabilities --action syncast.agent.delegate --disclosure full
syncast project-agent run syncast.project.inspect --input '{"limit":10}'
syncast project-agent run syncast.doc.graphql --input '{"query":"query { agents { agents { id name model allowLoadSkills skills { skillId skillType preload } childAgents { childAgentId alias displayName } } } }"}'
syncast project-agent run syncast.agent.delegate --input '{"goal":"整理项目方案","executor":{"kind":"model","model":"gemini-3.5-flash"},"wait":false}'
syncast project-agent run syncast.agent.followup --input '{"taskId":"<running-root-task-id>","prompt":"调整计划后继续完成剩余 TODO"}'
syncast project-agent run syncast.agent.thread.get --input '{"taskId":"<root-task-id>","subAgentId":"<thread-id>"}'
syncast project-agent run syncast.agent.thread.continue --input '{"taskId":"<root-task-id>","subAgentId":"<thread-id>","prompt":"沿用原上下文继续"}'
syncast project-agent materialize-media-segments --asset-id <audio-or-video-asset-id> --segments '[{"startTimeSeconds":0,"endTimeSeconds":15},{"startTimeSeconds":15,"endTimeSeconds":30}]'
```

`project-agent run` already forwards any capability discovered from the page,
so these Agent lifecycle actions do not require dedicated CLI subcommands or a
new bridge protocol. Default to `syncast.agent.followup` for a running root
task. Use `syncast.agent.thread.*` only for explicit child intervention or
recovery; moving a child to background requires
`confirmedContinuedBilling: true` and continues billing after the root answer.

Project document creation stays on the generic bridge; there is no separate
`syncast docs create` command. Create the complete tree and initial content in
one `ensureDocPages` mutation:

```shell
syncast project-agent run syncast.doc.graphql --input '{"query":"mutation EnsureDocs($inputs: [EnsureDocPageInput!]!) { ensureDocPages(inputs: $inputs) { success logicalKey docId created adopted contentWritten ready hasMeaningfulContent } }","variables":{"inputs":[{"logicalKey":"short-drama:episodes","title":"分集剧本","containerOnly":true},{"logicalKey":"short-drama:episode:01","title":"第一集","parentLogicalKey":"short-drama:episodes","initialMarkdown":"# 第一集\n\n## 场景一\n\n正文"}]}}'
```

Use a stable project-business `logicalKey` across retries and replay. For an
existing page, query its real id and pass `existingDocId`; never infer identity
from a possibly duplicated title. A content page requires meaningful
`initialMarkdown`, while only a directory may set `containerOnly: true`.
`success=true` plus `ready=true` is the write postcondition, so do not spend a
second tool call on a mechanical read. For one or many exact edits in an
existing body, prefer the structured action so the CLI does not have to invent
the Patch DSL:

```shell
syncast project-agent run syncast.docs.replaceText --input '{"docId":"doc-id","replacements":[{"search":"exact old text","replacement":"replacement text"}]}'
```

Every `search` must match exactly once in the latest Markdown; a missing or
ambiguous match rejects the whole batch without saving partial results. Use raw
`patchDoc` only for line, section, or whole-document operations, and first copy
the complete string format from `syncast.doc.graphql.explain`.
The full internal schema retains `createDocPage` for App and historical blank
page callers, but the external `syncast.doc.graphql` contract rejects it.

`project-agent` is mandatory whenever the user is working in a Syncast project,
including a new or otherwise empty project. Its capabilities are a
fail-closed set of high-level external actions. `syncast.imagine.submit` and
`syncast.imagine.submitToChannel` are public project-interaction actions because
they use the frontend queue and persist Imagine channel history.
`syncast.timeline.generationSlots.submit` is likewise public because it is the
Action equivalent of clicking a Slot's generate button.
`syncast.docs.imagineBlocks.*` exposes submit, history, selection, explicit
fix-as-asset, and restore as public equivalents of the document UI. Draft renderers,
lower-level Agent chat input, typed wait/result, notification actions, and
compatibility list/search/get aliases remain hidden. The absence of an explicit
channel-history request is not permission to leave the project flow: use
`syncast.imagine.submit` to resolve or create the project channel.

Media slicing is a project Asset operation, not a generation flag. Use
`syncast project-agent materialize-media-segments` (or generic `run
syncast.assets.materializeMediaSegments`) to turn explicit audio/video ranges
into normal project Assets. `--segments` accepts a non-empty JSON array or
`@file`; `--folder-id` is optional and otherwise follows the source Asset
placement. A full-source range reuses the source Asset without trimming, while
identical output content reuses an existing Asset without another create or
upload. Only distinct content creates a new Asset. The ordered result keeps
successful `assetId` values even if another segment fails. Use those returned
IDs as ordinary references in the next step. For a retryable external workflow,
pass a stable `--idempotency-key`; reusing that key with different input is an
error.

For deterministic delegated work, pass the action input `executor` explicitly:

- `{ "kind": "model", "model": "gemini-3.5-flash" }` starts from a direct model, which can discover every project Agent and can also create a fresh ad-hoc child.
- `{ "kind": "agent", "agentId": "<project-agent-id>" }` starts from that independent Agent declaration, which can name only its bound Agents and can also create a fresh ad-hoc child.

Bindings reference the same independent Agent declaration; they do not create a second child mode. Child runs are leaves, and fresh ad-hoc children do not inherit parent Agent instructions. Query the Agents GraphQL module before choosing `executor.agentId`. The CLI option `--agent-id` identifies the external operator and is unrelated to the project Agent selected by `executor.agentId`.

Always query and preserve `allowLoadSkills` together with `skills { skillId skillType preload }` when reading then updating an Agent. Every built-in Skill remains available on demand. Custom bindings identify the explicitly selected project Skills, while `preload` only controls startup instruction injection: `false` keeps a selected Skill available on demand and `true` injects its full instructions at startup. `allowLoadSkills: false` excludes only unselected project custom Skills; `true` expands discovery to the whole project custom-Skill catalog. Missing legacy `allowLoadSkills` and binding `preload` values are both interpreted as `true`, so new Agent declarations should write both booleans explicitly. Skill `depends` entries never use preload.

A delegated task uses the Agent/Skill snapshot captured at submission. Editing the project Agent or Skill changes the next task, not one already running. Child permissions can inherit or narrow the root task permission profile, but cannot exceed it.

Delegate only creative judgment, such as ideation, script design, or an
unknown structure. If the user supplied complete text, requested an exact
field edit, replacement, or specified removal, perform the deterministic write
directly instead of sending it through an internal Agent. Use project GraphQL
for new complete documents and metadata fields, `syncast.docs.replaceText` for
exact text edits in an existing document, and raw `patchDoc` for section or
removal operations. A same-scope, non-billable project write explicitly
requested by the user is already authorized. Ask again only for scope
expansion, additional credit spend, deletion, or an ambiguous target. Treat a
broad overwrite that would discard unrequested content as deletion, but do not
reconfirm an exact replacement the user already requested.

Before creating a custom Skill, query same-name Skills. Confirm `alwaysApply`,
typed `depends`, and whether an Agent binding is needed. After create/update,
read the complete Skill by its real id, validate all document references, and
inspect instructions, description, and metadata for external integration
leakage. If binding it to an Agent, preserve and re-read `allowLoadSkills` and
every `skills.preload` value.

`syncast project-agent wait --return-result` emits compact JSON by default:
final text, terminal task status, and artifact ids. Add `--full-result` only
when debugging message parts, thinking, or tool traces.

If an internal Syncast Agent pauses on an action approval, list pending approval
notifications:

```shell
syncast project-agent notifications \
  --type agent_action.approval_requested \
  --limit 5
```

Each approval notification includes `approvalId` and `respondAction`. The
external Agent may decide itself, or ask the human user first, then respond:

```shell
syncast project-agent approval respond <approvalId> --approve
syncast project-agent approval respond <approvalId> --deny --feedback "User rejected this action."
```

The CLI command calls `syncast.agent.approval.respond` in the connected project
page. It only succeeds for the external Agent identity that owns the delegated
internal Agent task; other external Agents receive `approval_actor_mismatch`.

To wait for a specific approval resolution notification, pass a type filter:

```shell
syncast project-agent wait \
  --ref '{"kind":"agent_action_approval","projectId":"...","taskId":"<approvalId>"}' \
  --type agent_action.approval_resolved
```

## Library and templates

When publishing Library packages, building project-template skeletons, or using
admin template commands, read
[references/library-and-templates.md](references/library-and-templates.md).

## Local file sync (legacy)

```shell
syncast sync
# or
syncast start
```

## Models and media inputs

When choosing models, handling reference media, or selecting an upscale route,
read [references/media-models.md](references/media-models.md). Project work
must still stay on the public Project Agent Actions described above.

## Troubleshooting

| Issue | Action |
|-------|--------|
| Not logged in | `syncast auth login` |
| Session expired | `syncast auth logout` then login again |
| 402 insufficient credits | Add credits in Syncast dashboard |
| Device code expired | Re-run `syncast auth login` |

## Configuration

Credentials: `~/.syncast/config.json`

Built-in environments:

| Env | API URL |
|-----|---------|
| `dev` (default) | `https://dev-syncast-service.latentnet.com` |
| `nightly` | `https://nightly-service.syncast.net` |
| `local` | `http://localhost:8901` |

Environment variables (override built-in):

- `SYNCAST_API_URL` or `API_URL` — API base URL

## Agent workflow summary

1. Determine whether the user is working in a Syncast project or wants only
   independent assets outside a project; ask if unclear.
2. For project work: connect with `syncast project-agent`, inspect the project,
   and generate through the appropriate public project Action. Never fall back
   to a direct generation command.
3. For explicitly standalone work: run `syncast auth login` / `status`, then
   choose by intent: `syncast imagine` for images, `syncast video` for video,
   `syncast audio` for complete audio/dubbing, `syncast music` for music, or
   `syncast sound-effect` for SFX, and consume the returned media URL.
