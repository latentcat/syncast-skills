---
name: syncast-cli
description: Install and use Syncast CLI to generate images and videos via Syncast AI services. Use when the user asks to install Syncast CLI, log in, run imagine/video generation, or integrate Syncast into an agent harness.
---

# Syncast CLI

Syncast CLI lets external agents call Syncast's image and video generation APIs from the terminal.

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
syncast imagine --prompt "a cat on the moon" --model nano-banana-pro --aspect-ratio 16:9
```

### Video

```shell
syncast video --prompt "a horse running on the beach" --model veo-3-1 --duration 8
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

For complex motion, multi-subject, or action-heavy scenes, prefer the Seedance 2.0 Global route `kittyvibe-seedance2.0global`. Use `kittyvibe-seedance2.0fastglobal` for faster/lower-cost Global previews.

## Project / Agent Action models

These model IDs require project assets or richer Imagine payloads. Use Syncast Agent Actions, project Imagine drafts, or the app UI rather than a bare `syncast imagine` / `syncast video` command:

For image cleanup after multi-round edits, use `recraft-ai/recraft-crisp-upscale`; it repairs noisy, scaly, grainy, or broken texture details without requiring a prompt.

For video upscaling, use `topaz/slp-2.5` when the source is AI-generated or modern footage that should become more realistic while preserving structure. Use `topaz/ast-2` when the user wants creative detail reconstruction or prompt-guided stylization. Topaz video upscalers support `target_resolution` values `1080p` and `4k`; Astra also supports `creativity`, `sharp`, `realism`, and optional `prompt`.

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
3. `syncast imagine` or `syncast video` — parse JSON output for `image_url` / `video_url`
4. Use URLs in downstream agent steps
