---
name: syncast-cli
description: Install and use Syncast CLI to generate images, videos, and sound effects via Syncast AI services. Use when the user asks to install Syncast CLI, log in, run imagine/video/sound-effect generation, or integrate Syncast into an agent harness.
---

# Syncast CLI

Syncast CLI lets external agents call Syncast's image, video, and sound-effect generation APIs from the terminal.

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

## Step 4: Generate

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

Use an existing remote filename without uploading again:

```shell
syncast imagine \
  --prompt "restyle this image" \
  --model nano-banana-2 \
  --reference-remote cli-reference/example.png
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

```shell
syncast imagine \
  --prompt "poster layout" \
  --input '{"references":[{"asset_id":"ref1","remote_filename":"uploads/ref.png","reference_type":"image"}]}'
```

### Video

```shell
syncast video --prompt "a horse running on the beach" --model veo-3-1 --duration 8
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

### Deliver an image into a Syncast project

Use the normal `imagine` command. This is the only public CLI generation entry
point; do not route generation through `project-agent`.

```shell
syncast imagine \
  --project <project-id> \
  --folder "/抖音复现/首帧" \
  --name "角色A首帧" \
  --prompt "Generate a cinematic character sheet."
```

- `--project` enables durable delivery into that project's Assets.
- `--name` sets the project asset display name.
- `--folder` accepts a folder name or `/`-separated path. Missing segments are created automatically when Syncast materializes the result. Use `/` or omit it for root.
- `--folder-id` is an advanced alternative when a verified current-project folder ID is already available. Do not combine it with `--folder`.
- `--project-id` remains a hidden compatibility alias for `--project`.
- Project-only fields in raw `--input` require `--project`; explicit CLI flags override raw delivery fields.

Treat `project_delivery.status: "queued"` as the successful handoff contract.
If generation succeeds but the response does not confirm project delivery, the
CLI still prints the generated URL and exits non-zero. Do not claim that an
asset was archived when delivery is missing or failed.

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
syncast project-agent run syncast.project.inspect --input '{"limit":20}'
syncast project-agent run syncast.doc.graphql --input '{"query":"query { agents { agents { id name model allowLoadSkills skills { skillId skillType preload } childAgents { childAgentId alias displayName } } } }"}'
syncast project-agent run syncast.agent.delegate --input '{"goal":"整理项目方案","executor":{"kind":"model","model":"gemini-3.5-flash"},"wait":false}'
```

`project-agent` is for work that genuinely depends on the opened project's live
Docs, Assets, timeline, or internal Agents. The CLI intentionally hides and
rejects `syncast.imagine.submit` / `submitToChannel`; use `syncast imagine
--project ...` for generation and folder delivery. This avoids a second,
browser-dependent generation API.

For deterministic delegated work, pass the action input `executor` explicitly:

- `{ "kind": "model", "model": "gemini-3.5-flash" }` starts from a direct model, which can discover every project Agent and can also create a fresh ad-hoc child.
- `{ "kind": "agent", "agentId": "<project-agent-id>" }` starts from that independent Agent declaration, which can name only its bound Agents and can also create a fresh ad-hoc child.

Bindings reference the same independent Agent declaration; they do not create a second child mode. Child runs are leaves, and fresh ad-hoc children do not inherit parent Agent instructions. Query the Agents GraphQL module before choosing `executor.agentId`. The CLI option `--agent-id` identifies the external operator and is unrelated to the project Agent selected by `executor.agentId`.

Always query and preserve `allowLoadSkills` together with `skills { skillId skillType preload }` when reading then updating an Agent. Every built-in Skill remains available on demand. Custom bindings identify the explicitly selected project Skills, while `preload` only controls startup instruction injection: `false` keeps a selected Skill available on demand and `true` injects its full instructions at startup. `allowLoadSkills: false` excludes only unselected project custom Skills; `true` expands discovery to the whole project custom-Skill catalog. Missing legacy `allowLoadSkills` and binding `preload` values are both interpreted as `true`, so new Agent declarations should write both booleans explicitly. Skill `depends` entries never use preload.

A delegated task uses the Agent/Skill snapshot captured at submission. Editing the project Agent or Skill changes the next task, not one already running. Child permissions can inherit or narrow the root task permission profile, but cannot exceed it.

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
| Image | `nano-banana-2`, `nano-banana-pro`, `gpt-image-2`, `oai-gpt-image-2` |
| Video | `kittyvibe-seedance2.0pro`, `kittyvibe-seedance2.0fast`, `kittyvibe-seedance2.0global`, `kittyvibe-seedance2.0fastglobal`, `veo-3-1`, `veo-3-1-fast`, `grok-video-3` |
| Audio | `fal-ai/elevenlabs/sound-effects/v2` |

Default image route: use `nano-banana-2`. For complex composition, long text, strict layout, or higher instruction-following needs, prefer `nano-banana-2` with `thinking_level: "High"` when using project Imagine / Agent Actions; with the direct CLI, still pass `--model nano-banana-2` and keep the prompt explicit. Use `nano-banana-pro` only when the user explicitly asks for Pro / Nano Banana Pro, or when the request needs a special style, strong stylization, or a specific art-direction exploration.

Treat image reference and image editing as one capability family: **image input**. If the user provides an existing image and asks to keep a character, reuse style, make a variation, change clothes, change background, improve layout, or edit the image, prefer an image-input-capable image model rather than switching mental models between "reference" and "edit". The default image-input route is still `nano-banana-2`; use `oai-gpt-image-2` when the user explicitly asks for OpenAI/GPT Image 2, mask editing, official API behavior, or strict control over `aspect_ratio`, `resolution`, `quality`, `output_format`, or `mask`.

For complex motion, multi-subject, or action-heavy scenes, prefer the Seedance 2.0 Global route `kittyvibe-seedance2.0global`. Use `kittyvibe-seedance2.0fastglobal` for faster/lower-cost Global previews.

Default video route: use `kittyvibe-seedance2.0pro` for text-to-video, image-to-video, and multi-reference/multimodal video. Do not switch to Veo, Grok, Vidu, HappyHorse, or Kling merely because the user provides images or says "references"; Seedance 2.0 is the default multimodal video route. Use `kittyvibe-seedance2.0fast` only for fast/low-cost previews, and use Global variants for complex action, multi-subject, high-motion, monster, fantasy, or difficult choreography requests.

For ambience, foley, UI clicks, impacts, creature sounds, rain, wind, or other non-musical sound design, prefer `syncast sound-effect` with `fal-ai/elevenlabs/sound-effects/v2`. Keep the final prompt in English; Chinese prompts may be auto-translated server-side, but agents should still write clean English sound-effect prompts by default.
Use `--duration-seconds`, `--prompt-influence`, and `--loop` for sound-effect controls. Do not pass or expose an output-format option for Eleven sound effects; Syncast pins the backend output format to MP3 44.1kHz / 128kbps.

## Media input capability

Direct `syncast imagine` supports image input without going through project Agent Actions. For local files on disk, use `--reference-image <path>`; the CLI uploads the file through Syncast's upload API and appends it to `task_request.input.references`. For files already in Syncast/R2, use `--reference-remote <remote_filename>`. Both flags are repeatable.

Use `--input <json>` or `--input-file <path>` to pass arbitrary image schema fields directly into `task_request.input`; these fields are merged with `model_type`, `prompt`, `aspect_ratio`, and `resolution`. CLI convenience flags `--width`, `--height`, and `--quality` write those schema fields directly, which is useful for custom dimensions such as `--resolution custom --width 2400 --height 3600`.

When a newer CLI exposes `syncast schema`, check it before building advanced payloads. The schema is the source of truth for which models accept image/video/audio input fields, limits, and examples. If `syncast schema` is unavailable or incomplete, use the direct passthrough flags above for known backend generation fields. Use Syncast Agent Actions, project Imagine drafts, or the app UI when the task depends on project-local asset selection, project Docs, asset naming, folder placement, UI-only workflows, or richer project context.

Stable conceptual mapping for agents:

| User intent | Capability | Default route |
|-------------|------------|---------------|
| Generate from text | Text-to-image | `syncast imagine --model nano-banana-2` |
| Use a local picture as character/style/layout/input | Image input | `syncast imagine --model nano-banana-2 --reference-image ./image.png` |
| Use an already uploaded picture | Image input | `syncast imagine --model nano-banana-2 --reference-remote <remote_filename>` |
| Edit an existing picture without a strict mask | Image input | `nano-banana-2` with `--reference-image` or `--reference-remote` |
| Mask edit / official OpenAI GPT Image 2 behavior | Image input + mask | `oai-gpt-image-2` |
| Generate video from text/images/multiple media | Multimodal video input | `kittyvibe-seedance2.0pro` |
| Fast video preview | Multimodal video input | `kittyvibe-seedance2.0fast` |
| Complex action / high motion / many subjects | Multimodal video input | `kittyvibe-seedance2.0global` |

Do not classify "image reference" and "image editing" as separate model families. They are both image-input tasks; choose the model by the required controls and then fill the schema-supported input fields.

## Project / Agent Action models

These routes depend on project-local asset selection, draft state, or UI-only workflows. Use Syncast Agent Actions, project Imagine drafts, or the app UI when the task needs those project contexts rather than a bare `syncast imagine` / `syncast video` command:

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

1. `syncast auth login` — user completes browser authorization
2. `syncast auth status` — confirm login
3. `syncast imagine`, `syncast video`, or `syncast sound-effect` — parse JSON output for `image_url` / `video_url` / `audio_url`
4. Use URLs in downstream agent steps
