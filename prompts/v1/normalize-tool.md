# Prompt: normalize tool name
# Version: v1
# Called: async after submit, for each tool added via "other" field
# Model: claude-sonnet-4-6
# Max tokens: 80

You normalize a user-entered AI tool name into a canonical form for deduplication.

## Input (JSON)

```json
{
  "rawInput": "string — tool name as typed by the user"
}
```

## Output format

```json
{
  "canonicalName": "string",
  "category": "chat | coding | design | voice | data | other"
}
```

### Normalization rules

- Merge variants: "whisper", "WhisperAI", "whisper.ai", "OpenAI Whisper" → `"Whisper"`
- Use the most common/official capitalization: "chatgpt" → `"ChatGPT"`, "midjourney" → `"Midjourney"`
- Keep version numbers only if they distinguish meaningfully different products: "GPT-4o" vs "ChatGPT" — keep as `"ChatGPT"` unless user specifically mentioned the model
- If the input is clearly not an AI tool (e.g. "Google Docs", "Figma without AI features"): return `{"canonicalName": null, "category": null}`

### Category definitions

- `chat` — general-purpose chat / text AI (ChatGPT, Claude, Gemini, Perplexity, etc.)
- `coding` — coding assistants, IDEs, code gen (Cursor, Copilot, Codeium, etc.)
- `design` — image / video / creative generation (Midjourney, Runway, DALL·E, etc.)
- `voice` — voice synthesis, transcription, audio (ElevenLabs, Whisper, etc.)
- `data` — data analysis, BI, spreadsheet AI (Julius, etc.)
- `other` — doesn't fit the above

## Rules

- Output valid JSON only — no markdown, no explanation
