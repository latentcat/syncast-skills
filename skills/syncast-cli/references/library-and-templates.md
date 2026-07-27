# Library and template reference

## Contents

- [Library resource packages](#library-resource-packages)
- [Project template skeletons](#project-template-skeletons)
- [Admin template management](#admin-template-management)
- [Template categories](#template-categories)

## Library resource packages

Publish normal reusable resources through `syncast library publish`, not the
admin-only `syncast template` commands. This route covers local packages,
private shares, team libraries, and public-review submissions.

Ask the CLI for the current machine-readable contract before creating a
package:

```shell
syncast library guide
syncast library guide agent
syncast library guide skill
syncast library guide project_template
```

The guide is local and needs no login. Treat its JSON as authoritative;
`syncast library publish --help` and `syncast library import --help` point back
to it.

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

Use snake_case in new Library manifests. The CLI accepts documented camelCase
aliases for programmatic callers. `preload` belongs only to an Agent `skills`
binding and must not appear in Skill `depends`. Include/install every referenced
custom Skill and child Agent before applying an Agent package.

Set `allow_load_skills` explicitly to `false` unless the Agent intentionally
needs the full project custom-Skill catalog. `false` still allows all built-in
Skills, selected custom Skills, their dependencies, and `always_apply` custom
Skills. A missing legacy value remains `true`; this field does not replace
`preload`.

```shell
# Personal or team library item
syncast library publish ./bundle.json --type project_template --target personal
syncast library publish ./bundle.json --type project_template --target team:<teamId>

# Public template review submission
syncast library publish ./bundle.json --type project_template --target public \
  --source-project-id <projectId>
```

`project_template` is the product alias for stored type
`project_template_bundle`. Personal/team targets keep the package in workspace
library item `data`; public targets send the same package through
`typedData.content` for review.

## Project template skeletons

Keep reusable Docs and resource directories inside the package content:

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
`standard_project_template` is accepted. Do not split skeleton Docs or resource
folders into ad hoc sidecar files; import/export/share/public submission must
preserve them inside the package.

## Admin template management

All `syncast template` commands require admin privileges.

```shell
# List/search/fetch all
syncast template list
syncast template list --category image_prompt
syncast template list --search "steampunk"
syncast template list --all
syncast template categories

# Create
syncast template create \
  --id "my-template-slug" \
  --category image_prompt \
  --title "My Template" \
  --title-zh "我的模板" \
  --description "A description" \
  --content '{"prompt":"a cat on the moon","negative_prompt":"blurry"}' \
  --tags "concept_art,fantasy" \
  --thumbnail "https://example.com/thumb.jpg"

# Update (creates a new version)
syncast template update \
  --id "my-template-slug" \
  --title "Updated Title" \
  --tags "concept_art,sci-fi"

# Hide / restore / revisions
syncast template hide --id "my-template-slug"
syncast template hide --id "my-template-slug" --unhide
syncast template revisions --id "my-template-slug"

# Bulk validation/upload (`-f` is reserved for output format)
syncast template bulk-upload --file ./templates.json --dry-run
syncast template bulk-upload --file ./templates.json
```

Bulk JSON is an array with required `template_id`, `category`, and `title`;
optional fields include `title_zh`, `description`, `description_zh`, `content`,
`tags`, and `thumbnail_url`.

## Template categories

| Category | Description |
| --- | --- |
| `character` | Character templates (persona, appearance) |
| `image_prompt` | Image generation prompt templates |
| `agent_prompt` | Agent system prompt templates |
| `custom_agent` | User or team Agent package submitted for review |
| `custom_skill` | User or team Skill package submitted for review |
| `imagine_optimize_preset` | Imagine prompt optimization preset |
| `doc_spec` | Single project document/spec template |
| `project_spec_bundle` | Bundle of project specs and related setup |
| `project_template_bundle` | Full project template, optionally with `standardProjectTemplate` |
