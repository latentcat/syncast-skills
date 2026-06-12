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

## Task management

```shell
# Stream / wait for an existing task
syncast task status <task_id>

# Cancel
syncast task cancel <task_id>
```

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

When a newer CLI exposes `syncast schema`, check it before building advanced payloads. The schema is the source of truth for which models accept image/video/audio input fields, limits, and examples. If `syncast schema` is unavailable or incomplete, use the direct passthrough flags above for known backend schema fields; use Syncast Agent Actions, project Imagine drafts, or the app UI only when the task depends on project-local asset selection, UI-only workflows, or richer project context.

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
