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

Document Imagine blocks are a third project-local contract. They must use the
Action Bridge so the generated asset is written back to the originating block:

```shell
syncast project-agent capabilities --action syncast.docs.imagineBlocks.submitBatch --disclosure full
syncast project-agent run syncast.docs.imagineBlocks.submitBatch \
  --input '{"docId":"<doc-id>"}'
```

Use `syncast.docs.imagineBlocks.submit` with `{ docId, blockId }` for one card.
`submitBatch` accepts optional `blockIds`; omitting them is equivalent to the
document's “generate all pending” button. It starts each block submission in
parallel, isolates per-block failures, and returns one task `ref` per successful
block. Direct `syncast imagine`, even with `--project`, cannot replace this
contract because it does not update document block state.

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
syncast project-agent materialize-media-segments --asset-id <audio-or-video-asset-id> --segments '[{"startTimeSeconds":0,"endTimeSeconds":15},{"startTimeSeconds":15,"endTimeSeconds":30}]'
```

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
second tool call on a mechanical read. Use `patchDoc` for an existing body.
The full internal schema retains `createDocPage` for App and historical blank
page callers, but the external `syncast.doc.graphql` contract rejects it.

`project-agent` is mandatory whenever the user is working in a Syncast project,
including a new or otherwise empty project. Its capabilities are a
fail-closed set of high-level external actions. `syncast.imagine.submit` and
`syncast.imagine.submitToChannel` are public project-interaction actions because
they use the frontend queue and persist Imagine channel history.
`syncast.timeline.generationSlots.submit` is likewise public because it is the
Action equivalent of clicking a Slot's generate button.
`syncast.docs.imagineBlocks.submit` and `submitBatch` are public equivalents of
the document card and top-level batch buttons. Draft renderers,
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
field edit, replacement, or specified removal, use project GraphQL directly;
do not send the text through an internal Agent for rewriting or mechanical
copying. A same-scope, non-billable project write explicitly requested by the
user is already authorized. Ask again only for scope expansion, additional
credit spend, deletion, or an ambiguous target. Treat a broad overwrite that
would discard unrequested content as deletion, but do not reconfirm an exact
replacement the user already requested.

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

## Library resource packages

Normal users publish reusable resources through `syncast library publish`, not
the admin-only `syncast template` commands. This is the route used by local
resource packages, private shares, team libraries, and public-review
submissions.

Before creating a package, ask the CLI for the current machine-readable field guide and canonical example:

```shell
syncast library guide
syncast library guide agent
syncast library guide skill
syncast library guide project_template
```

The guide is local and does not require login. Use its JSON output as the contract for package creation; `syncast library publish --help` and `syncast library import --help` point back to it.

Canonical Agent Skill binding:

```json
{
  "type": "agent",
  "id": "story-planner",
  "name": "Story Planner",
  "instructions": "Read the project spec, then return a structured plan.",
  "allow_load_skills": false,
  "skills": [
    { "skill_id": "docs", "skill_type": "builtin", "preload": false },
    {
      "skill_id": "continuity-checker",
      "skill_type": "custom",
      "preload": true
    }
  ],
  "child_agents": [
    {
      "child_agent_id": "shot-planner",
      "alias": "shots",
      "when_to_use": "Use for shot breakdown.",
      "handoff_contract": "Return shot ids, risks, and next actions.",
      "project_spec_strategy": "inherit"
    }
  ]
}
```

Canonical custom Skill dependency declaration:

```json
{
  "type": "skill",
  "id": "continuity-checker",
  "name": "Continuity Checker",
  "instructions": "Check character and shot continuity.",
  "always_apply": false,
  "depends": [
    { "skill_id": "docs", "skill_type": "builtin" },
    { "skill_id": "visual-style", "skill_type": "custom" }
  ]
}
```

Use snake_case in new Library manifests. The CLI also accepts documented camelCase aliases for programmatic callers. `preload` belongs only to an Agent `skills` binding; it must not appear in a Skill `depends` item. Agent packages reference custom Skills and child Agents by id, so include/install all dependencies before applying them to a project.

`allow_load_skills` is a compatibility field for whether an Agent may discover and load **unselected project custom Skills**. New packages should set it explicitly to `false` unless the Agent intentionally needs the full project custom-Skill catalog. `false` still allows every built-in Skill, selected custom Skills, their dependencies, and `always_apply` custom Skills. A missing legacy value remains `true`. This field does not replace `preload`: `preload` only controls which selected Skills inject their full instructions at startup.

```shell
# Personal or team library item
syncast library publish ./bundle.json --type project_template --target personal
syncast library publish ./bundle.json --type project_template --target team:<teamId>

# Public template review submission
syncast library publish ./bundle.json --type project_template --target public \
  --source-project-id <projectId>
```

`project_template` is the product alias for the stored
`project_template_bundle` type. Personal and team targets write the complete
package into workspace library item `data`. Public targets send the same package
through `typedData.content`, and the backend preserves it into the reviewed
template content.

Project template bundles may carry ordinary bundle resources plus a reusable
project skeleton:

```json
{
  "type": "project_template",
  "id": "short-drama-skeleton",
  "name": "Short Drama Skeleton",
  "agents": [{ "id": "writer" }],
  "skills": [{ "id": "continuity-checker" }],
  "promptTemplates": [],
  "projectSpecs": [],
  "standardProjectTemplate": {
    "schemaVersion": 1,
    "docs": [
      {
        "id": "story-bible",
        "title": "Story Bible",
        "body": "# Story Bible\n\nProduction rules."
      },
      {
        "id": "episode-outline",
        "parentId": "story-bible",
        "title": "Episode Outline",
        "body": "Scene beats."
      }
    ],
    "resourceDirs": [
      { "path": "References/Characters" },
      { "path": "Shots/Act 1" }
    ]
  }
}
```

Use camel-case `standardProjectTemplate` for new packages. Legacy
`standard_project_template` is accepted, but agents should not split skeleton
Docs or resource folders into ad hoc sidecar files. Import/export/share/public
submission routes must keep the skeleton inside the package content so other
clients can discover and consume it.

## Template management (admin only)

All template commands require admin privileges. Non-admin users will receive a permission error.

### List templates

```shell
# List all templates (including hidden), paginated
syncast template list

# Filter by category
syncast template list --category image_prompt

# Search by text
syncast template list --search "steampunk"

# Fetch all pages at once
syncast template list --all
```

### List categories

```shell
syncast template categories
```

### Create a template

```shell
syncast template create \
  --id "my-template-slug" \
  --category image_prompt \
  --title "My Template" \
  --title-zh "我的模板" \
  --description "A description" \
  --content '{"prompt":"a cat on the moon","negative_prompt":"blurry"}' \
  --tags "concept_art,fantasy" \
  --thumbnail "https://example.com/thumb.jpg"
```

### Update a template (creates new version)

```shell
syncast template update \
  --id "my-template-slug" \
  --title "Updated Title" \
  --tags "concept_art,sci-fi"
```

### Hide / unhide a template

```shell
# Soft-delete (hide)
syncast template hide --id "my-template-slug"

# Restore (unhide)
syncast template hide --id "my-template-slug" --unhide
```

### View version history

```shell
syncast template revisions --id "my-template-slug"
```

### Bulk upload from JSON file

```shell
# Validate without uploading
syncast template bulk-upload --file ./templates.json --dry-run

# Upload (note: use --file, not -f which is reserved for --format)
syncast template bulk-upload --file ./templates.json
```

The JSON file must be an array of objects with required fields `template_id`, `category`, `title` and optional fields `title_zh`, `description`, `description_zh`, `content`, `tags`, `thumbnail_url`.

### Template categories

| Category | Description |
|----------|-------------|
| `character` | Character templates (persona, appearance) |
| `image_prompt` | Image generation prompt templates |
| `agent_prompt` | Agent system prompt templates |
| `custom_agent` | User or team Agent package submitted for review |
| `custom_skill` | User or team Skill package submitted for review |
| `imagine_optimize_preset` | Imagine prompt optimization preset |
| `doc_spec` | Single project document/spec template |
| `project_spec_bundle` | Bundle of project specs and related setup |
| `project_template_bundle` | Full project template bundle, optionally including `standardProjectTemplate` |

## Local file sync (legacy)

```shell
syncast sync
# or
syncast start
```

## Common direct CLI models

| Media | Models |
|-------|--------|
| Image | `nano-banana-2`, `nano-banana-pro`, `seedream-5-0-pro`, `gpt-image-2`, `oai-gpt-image-2` |
| Video | `kittyvibe-seedance2.0pro`, `kittyvibe-seedance2.0fast`, `kittyvibe-seedance2.0global`, `kittyvibe-seedance2.0fastglobal`, `veo-3-1`, `veo-3-1-fast`, `grok-video-3` |
| Complete audio / scene-aware dubbing | `bytedance/seed-audio-1.0` |
| Music | `zhenzhen-suno-v5.5` (default), `lyria-3-clip`, `lyria-3-pro` (explicit alternatives) |
| Sound effects | `fal-ai/elevenlabs/sound-effects/v2` |
| Clean TTS / voice setup | `minimax/speech-2.8-hd`, `minimax/speech-2.8-turbo`, `minimax/voice-clone`, `minimax/voice-design` |

Default image route: use `nano-banana-2`. For complex composition, long text, strict layout, or higher instruction-following needs, an internal Syncast Agent may use the model schema's `thinking_level: "High"`; with the direct CLI, pass `--model nano-banana-2` and keep the prompt explicit. Use `nano-banana-pro` only when the user explicitly asks for Pro / Nano Banana Pro, or when the request needs a special style, strong stylization, or a specific art-direction exploration.

Treat image reference and image editing as one capability family: **image input**. If the user provides an existing image and asks to keep a character, reuse style, make a variation, change clothes, change background, improve layout, or edit the image, prefer an image-input-capable image model rather than switching mental models between "reference" and "edit". The default image-input route is still `nano-banana-2`; use `oai-gpt-image-2` when the user explicitly asks for OpenAI/GPT Image 2, mask editing, official API behavior, or strict control over `aspect_ratio`, `resolution`, `quality`, `output_format`, or `mask`.

For complex motion, multi-subject, or action-heavy scenes, prefer the Seedance 2.0 Global route `kittyvibe-seedance2.0global`. Use `kittyvibe-seedance2.0fastglobal` for faster/lower-cost Global previews.

Default video route: use `kittyvibe-seedance2.0pro` for text-to-video, image-to-video, and multi-reference/multimodal video. Do not switch to Veo, Grok, Vidu, HappyHorse, or Kling merely because the user provides images or says "references"; Seedance 2.0 is the default multimodal video route. Use `kittyvibe-seedance2.0fast` only for fast/low-cost previews, and use Global variants for complex action, multi-subject, high-motion, monster, fantasy, or difficult choreography requests.

The retired CLI aliases `seedance2.0pro` and `seedance2.0fast` are accepted only
for compatibility and are rewritten to `kittyvibe-seedance2.0pro` and
`kittyvibe-seedance2.0fast`. Agents should emit the KittyVibe names directly.

Default complete-audio route: use `syncast audio` with `bytedance/seed-audio-1.0` for scene-aware dubbing, dialogue, narration, radio drama, multi-character performance, or any audio piece that mixes voices with ambience, effects, or music. Use MiniMax Speech instead when the task is clean, stable, reusable long-form TTS without scene mixing.

Default music route: use `syncast music` with `zhenzhen-suno-v5.5` for songs, instrumentals, BGM, scores, lyrics, extensions, and covers. Use `--mode custom` for final lyrics or lyric generation, and `--mode extend` / `cover` with one real audio reference. Lyria is an explicit alternative only when the user names Lyria/Google music generation or specifically needs image-inspired music.

For ambience, foley, UI clicks, impacts, creature sounds, rain, wind, or other non-musical sound design, prefer `syncast sound-effect` with `fal-ai/elevenlabs/sound-effects/v2`. Keep the final prompt in English; Chinese prompts may be auto-translated server-side, but agents should still write clean English sound-effect prompts by default.
Use `--duration-seconds`, `--prompt-influence`, and `--loop` for sound-effect controls. Do not pass or expose an output-format option for Eleven sound effects; Syncast pins the backend output format to MP3 44.1kHz / 128kbps.

## Media input capability

For standalone assets, direct `syncast imagine` accepts local image input with
repeatable `--reference-image <path>`. For project work, resolve the source to a
real project Asset ID and pass it through the project Action's `references` or
model-specific input. If a required local file is not yet in the project, ask
the user to import it into the intended project instead of silently switching
to standalone generation.

Use `--input <json>` or `--input-file <path>` to pass arbitrary image schema fields directly into `task_request.input`; these fields are merged with `model_type`, `prompt`, `aspect_ratio`, and `resolution`. CLI convenience flags `--width`, `--height`, and `--quality` write those schema fields directly, which is useful for custom dimensions such as `--resolution custom --width 2400 --height 3600`.

Use `syncast imagine --help` only for the standalone image route. In project
work, query the public `syncast.imagine.models` Action for current model limits,
then submit through `syncast.imagine.submit` or the more specific project
Action. Do not assume a nonexistent `syncast schema` command, and do not use raw
direct-CLI passthrough to escape the project workflow.

Stable conceptual mapping for agents:

| User intent | Capability | Standalone route | Project route |
|-------------|------------|------------------|---------------|
| Generate from text | Text-to-image | `syncast imagine --model nano-banana-2` | `syncast.imagine.submit` |
| Use a local picture as character/style/layout/input | Image input | `syncast imagine --reference-image ./image.png` | Import it as a project Asset, then use its ID with `syncast.imagine.submit` |
| Use an image already in a Syncast project | Image input | Not a standalone task | Project Asset ID in `syncast.imagine.submit` |
| Edit an existing picture without a strict mask | Image input | `nano-banana-2` with `--reference-image` | `nano-banana-2` through `syncast.imagine.submit` |
| Mask edit / official OpenAI GPT Image 2 behavior | Image input + mask | `oai-gpt-image-2` | `oai-gpt-image-2` through `syncast.imagine.submit` |
| Generate video from text/images/multiple media | Multimodal video input | `kittyvibe-seedance2.0pro` | `kittyvibe-seedance2.0pro` through `syncast.imagine.submit` |
| Fast video preview | Multimodal video input | `kittyvibe-seedance2.0fast` | `kittyvibe-seedance2.0fast` through `syncast.imagine.submit` |
| Complex action / high motion / many subjects | Multimodal video input | `kittyvibe-seedance2.0global` | `kittyvibe-seedance2.0global` through `syncast.imagine.submit` |

Do not classify "image reference" and "image editing" as separate model families. They are both image-input tasks; choose the model by the required controls and then fill the schema-supported input fields.

## Project-context workflows

If the user is working in a project, stay on Syncast Agent Actions for the
entire workflow even after the prompt, model, references, and destination are
known. Use `syncast.imagine.submit` by default,
`syncast.imagine.submitToChannel` for a specified existing channel,
`syncast.docs.imagineBlocks.submit/submitBatch` for document cards, and
`syncast.timeline.generationSlots.submit` for timeline Slots. Direct generation
commands are only for the separately confirmed external-asset route.

For image cleanup after multi-round edits, use `recraft-ai/recraft-crisp-upscale`; it repairs noisy, scaly, grainy, or broken texture details without requiring a prompt.

For video upscaling, use `topaz/slp-2.5` when the source is AI-generated or modern footage that should become more realistic while preserving structure. Use `fal-ai/topaz/upscale/video` when the user explicitly wants the fal Starlight Precise 2.5 route or needs fal parameters such as `upscale_factor`, `target_fps`, `compression`, `noise`, `halo`, `grain`, `recover_detail`, or `H264_output`. Use `topaz/ast-2` when the user wants creative detail reconstruction or prompt-guided stylization. Topaz video upscalers support `target_resolution` values `1080p` and `4k`; the fal route may also accept an explicit `upscale_factor` from 1 to 4; Astra also supports `creativity`, `sharp`, `realism`, and optional `prompt`.

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
