# Media models and input reference

Read this reference when choosing a direct-generation model, handling media
input, or selecting a project generation model. The mandatory project versus
standalone routing in the parent skill still takes precedence.

## Model defaults

| Media | Models |
| --- | --- |
| Image | `nano-banana-2`, `nano-banana-pro`, `seedream-5-0-pro`, `gpt-image-2`, `oai-gpt-image-2` |
| Video | `kittyvibe-seedance2.0pro`, `kittyvibe-seedance2.0fast`, `kittyvibe-seedance2.0global`, `kittyvibe-seedance2.0fastglobal`, `veo-3-1`, `veo-3-1-fast`, `grok-video-3` |
| Complete audio / dubbing | `bytedance/seed-audio-1.0` |
| Music | `zhenzhen-suno-v5.5` (default), `lyria-3-clip`, `lyria-3-pro` (explicit alternatives) |
| Sound effects | `fal-ai/elevenlabs/sound-effects/v2` |
| Clean TTS / voice setup | `minimax/speech-2.8-hd`, `minimax/speech-2.8-turbo`, `minimax/voice-clone`, `minimax/voice-design` |

Use `nano-banana-2` by default for image generation and image input. Use
`nano-banana-pro` only when explicitly requested or for special stylization/art
direction. Use `oai-gpt-image-2` for an explicit OpenAI/GPT Image 2 request,
mask editing, official API behavior, or strict `aspect_ratio`, `resolution`,
`quality`, `output_format`, or `mask` control.

Treat image reference and image editing as one image-input capability family.

Use `kittyvibe-seedance2.0pro` by default for text-to-video, image-to-video, and
multi-reference video. Use `kittyvibe-seedance2.0fast` for fast previews and
the Global variants for complex action, many subjects, high motion, monsters,
fantasy, or difficult choreography. Retired aliases `seedance2.0pro` and
`seedance2.0fast` are compatibility input only; emit KittyVibe names.

Use `bytedance/seed-audio-1.0` for scene-aware dialogue, narration, radio drama,
or complete mixes with ambience/SFX/music. Use MiniMax Speech for clean,
reusable long-form TTS. Use `zhenzhen-suno-v5.5` for songs, instrumentals, BGM,
lyrics, extensions, and covers; choose Lyria only when explicitly requested.
Use `fal-ai/elevenlabs/sound-effects/v2` for ambience, foley, UI sounds, impacts,
creatures, rain, and wind. Write the final SFX prompt in English and do not
expose output format; Syncast pins MP3 44.1kHz/128kbps.

## Media input

Standalone image input uses repeatable `--reference-image <path>`. Project work
uses real project Asset IDs in the relevant public Action. Ask the user to
import a required local file into the intended project instead of silently
switching to standalone generation.

Use `--input <json>` or `--input-file <path>` for arbitrary direct image schema
fields. `--width`, `--height`, and `--quality` write those schema fields. Use
`syncast imagine --help` only for standalone work; in projects query
`syncast.imagine.models`, then submit through a project Action.

| User intent | Standalone route | Project route |
| --- | --- | --- |
| Text-to-image | `syncast imagine --model nano-banana-2` | `syncast.imagine.submit` |
| Local picture as input | `--reference-image ./image.png` | Import, then pass its Asset ID |
| Existing project image | Not standalone | Project Asset ID in `syncast.imagine.submit` |
| Image edit without mask | `nano-banana-2` + reference image | `nano-banana-2` project submit |
| Mask / official GPT Image 2 | `oai-gpt-image-2` | `oai-gpt-image-2` project submit |
| Multimodal video | `kittyvibe-seedance2.0pro` | Same model through project submit |
| Fast video preview | `kittyvibe-seedance2.0fast` | Same model through project submit |
| Complex/high-motion video | `kittyvibe-seedance2.0global` | Same model through project submit |

## Project workflows and upscaling

Stay on Project Agent Actions for the entire project workflow. Use
`syncast.imagine.submit` by default, `submitToChannel` for a specified existing
channel, `syncast.docs.imagineBlocks.*` for document versions, and
`syncast.timeline.generationSlots.submit` for timeline Slots.

For image cleanup after iterative edits, use
`recraft-ai/recraft-crisp-upscale`. For video realism-preserving upscale use
`topaz/slp-2.5`; use `fal-ai/topaz/upscale/video` when explicit fal/Starlight
controls are required, and `topaz/ast-2` for creative reconstruction or
prompt-guided stylization. Topaz routes support `target_resolution` 1080p/4k;
the fal route may accept `upscale_factor` 1–4, and Astra also exposes
`creativity`, `sharp`, `realism`, and optional `prompt`.
