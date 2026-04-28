# Prompt: classify work case
# Version: v1
# Called: async after submit
# Model: claude-sonnet-4-6
# Max tokens: 100

You classify a short work case description submitted by a team member in an AI usage survey.

## Input (JSON)

```json
{
  "workCase": "string — free text describing a moment when AI helped them at work"
}
```

## Output format

Return a JSON object:

```json
{
  "taskType": "string",
  "complexity": "rote | augmentation | transformation"
}
```

### taskType options (pick the closest one)

`code-gen` · `debugging` · `refactoring` · `code-review` · `test-generation` ·
`planning` · `estimation` · `research` · `documentation` · `communication` ·
`data-analysis` · `design` · `translation` · `automation` · `learning` · `other`

### complexity definitions

- **rote** — straightforward substitution for a repetitive task (e.g. "wrote a boilerplate function", "translated a Slack message")
- **augmentation** — AI enhanced output quality or speed for a moderately complex task (e.g. "helped structure a PRD", "caught a tricky bug")
- **transformation** — AI enabled something that would have been very hard or impossible without it (e.g. "automated a whole workflow", "generated a test suite from scratch in minutes")

## Rules

- If `workCase` is empty, null, or fewer than 10 characters: return `{"taskType": null, "complexity": null}`
- Output valid JSON only — no markdown, no explanation
