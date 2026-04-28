# AI-Pulse — Scoring & Analytics

## Overview

Each survey response produces a set of numeric metrics, an **AI-Native Score** (0–100, analytics only), and an **archetype** shown to the respondent. Open-text fields are classified asynchronously via Claude API using versioned prompts from `prompts/v1/`.

Scoring runs synchronously in `lib/scoring.ts` on submit. Case complexity is populated asynchronously after `classify-case` runs — the score is updated in MongoDB once classification completes.

---

## Numeric Metrics

Computed directly from form selections, no AI involved.

| Metric | Source | In Score? |
|---|---|---|
| **Frequency Score** | Section 1 | ✅ 35% |
| **Depth Score** | Section 3 | ✅ 30% |
| **Case Complexity** | classify-case (async) | ✅ 25% |
| **Sharing Propensity** | Section 4 | ✅ 10% |
| **Stack Breadth** | Section 1 | descriptive only |
| **Dropped Tools Rate** | Section 1 | descriptive only |
| **Personal Investment** | Section 1 | descriptive only |
| **Want Coverage Count** | Section 1 | descriptive only |
| **Use Case Diversity** | Section 2 | descriptive only |
| **Friction Score** | Section 5 | descriptive only |
| **Demand Signal** | Section 5 | standalone (see below) |
| **Agent Fill Rate** | Final | aggregate only |

---

## Frequency Score (0–4)

**Average** frequency across all tools with `status === "use"`, not the maximum.

| Frequency selected | Points |
|---|---|
| Multiple times a day | 4 |
| A few times a week | 3 |
| Once a week or less | 2 |
| Just tried it | 1 |

```ts
frequencyScore = usedTools.length > 0
  ? usedTools.reduce((sum, t) => sum + t.frequencyPoints, 0) / usedTools.length
  : 0
```

Range: 0–4. A daily single-tool user and a daily multi-tool user both reflect high frequency — but Stack Breadth (descriptive) shows the difference in context.

---

## Depth Score (0–10)

```
usesSavedPrompts          ? 3   : 0    // role-neutral, raised from 2
+ min(savedPromptsCount, 5) × 0.4     // up to 2 bonus points
+ usesApiIntegrations     ? 2.5 : 0   // lowered from 3
+ usesAiInEditor          ? 2.5 : 0   // lowered from 3
```

`usesAiInEditor` is derived from Section 1: any tool with `accessMethods` containing `"IDE integration"` and `status === "use"`.

| Profile | Max reachable |
|---|---|
| Non-developer (no IDE, no API) | 5 / 10 |
| Developer, no saved prompts | 5 / 10 |
| Developer, full depth | 10 / 10 |

IDE integration and API are still meaningful signals but no longer dominate. Saved prompts are the highest single-item signal — they reflect intentional, repeatable practice regardless of role.

---

## Case Complexity (1–3)

Populated by `classify-case.md` after submit. Written to `computed.caseComplexity`.

| Value | Points | Description |
|---|---|---|
| `rote` | 1 | Template use, single-step tasks |
| `augmentation` | 2 | AI speeds up or improves existing workflow |
| `transformation` | 3 | AI enables something that wasn't done before |

Score uses the normalized value: `caseComplexity / 3`.

If classification hasn't run yet (async lag), Score is computed without this component and recalculated once it arrives.

---

## Sharing Propensity (0–3)

```
sharingPreference === "yes"   → 2
sharingPreference === "anon"  → 1
sharingPreference === "no"    → 0
+ workCaseFileUrl !== null    → +1
```

---

## AI-Native Score (0–100)

```ts
aiNativeScore = Math.round((
  (frequencyScore / 4)          * 0.35 +
  (depthScore / 10)             * 0.30 +
  (caseComplexity / 3)          * 0.25 +   // 0 until async classifier runs
  (sharingPropensity / 3)       * 0.10
) * 100)
```

**Not shown to respondents.** Score is recalculated when `caseComplexity` is populated.

---

## Archetypes

Shown to the respondent on the summary screen.

| Archetype | Score | Description |
|---|---|---|
| 🦕 **Explorer** | 0–25 | Occasional use, no systematic approach |
| 🚀 **Practitioner** | 26–55 | Regular use, task-driven, 1–2 tools in focus |
| ⚡ **Integrator** | 56–80 | Broad use, automations, AI changes how they work |
| 🧠 **Ambassador** | 81–100 | Everything above, plus actively shares with the team |

---

## Demand Signal (0–2) — standalone metric

Removed from AI-Native Score. High demand signal means current low fluency, not high AI-nativeness. Tracked separately as an **input for manager action**.

```
wantsGuidance === true              → +1
inp-guidance value length > 10      → +1
```

Used in analytics for: "who needs help and with what specifically."

---

## Descriptive Metrics (not in Score)

These appear in analytics and inform decisions — they are not components of AI-Native Score.

| Metric | Why it's descriptive, not scored |
|---|---|
| **Stack Breadth** | More tools ≠ deeper use. 8 tools tried once < 2 tools used daily |
| **Use Case Diversity** | Checkbox count doesn't reflect depth within each use case |
| **Friction Score** | Captures pain, not capability |
| **Dropped Tools Rate** | Useful signal for tool decisions, not personal AI-nativeness |
| **Personal Investment** | Input for subscription decisions, not a score component |
| **Want Coverage Count** | Input for budget decisions |

---

## Claude API Classifiers

Run **asynchronously** after the response is saved to MongoDB. Results written back to `computed.*`.

Prompts are frozen in `prompts/v1/`. Do not modify them — classifiers must be identical across iterations for results to be comparable. To update: create `prompts/v2/`, document the change.

| Prompt file | Field | Output |
|---|---|---|
| `archetype.md` | `computed.summaryCard` | Short prose card shown to respondent (sync) |
| `classify-case.md` | `computed.caseType`, `computed.caseComplexity` | Task domain + `rote \| augmentation \| transformation` |
| `classify-pain.md` | `computed.painType` | `model-quality \| interface \| cost \| security \| skill` |
| `classify-skills.md` | `computed.skillsDomains` | `code \| writing \| analysis \| design \| other` |
| `normalize-tool.md` | `tools` collection | Canonical name + category; shown to others after 2+ mentions |

---

## Cross-Iteration Tracking

The survey repeats every 1–2 months. Core questions stay identical. These are the signals that indicate real change:

| Signal | What it shows |
|---|---|
| AI-Native Score delta | Is the team's AI fluency growing? |
| Archetype distribution shift | Are Explorers becoming Practitioners? |
| `try → use` conversion | Did last round's "want to try" turn into active use? |
| caseComplexity distribution | Is the quality of AI use improving (rote → transformation)? |
| Demand Signal specificity | Vague → specific = fluency growth |
| Dropped tools + reason | "Found something better" is healthy; "didn't understand" is a flag |
| Agent Fill Rate | Proxy for agentic maturity across the team |

**What can evolve between iterations:** open question wording, tool list, design.
**What must stay fixed:** tool matrix structure, task checkboxes, prompts yes/no, API checkbox, friction scale, classifier prompts.

---

## Analytics Views (planned `/admin`)

| View | Query |
|---|---|
| Archetype breakdown | `GROUP BY computed.archetype` |
| Score distribution | histogram of `computed.aiNativeScore` |
| Top tools by usage | `WHERE status="use" GROUP BY toolName ORDER BY count DESC` |
| Dropped tools + reasons | `WHERE status="dropped"`, aggregate `dropReason` |
| Subscription candidates | `WHERE payment="want-coverage" OR wantCoverage=true GROUP BY toolName` |
| Pain type distribution | `GROUP BY computed.painType` |
| Guidance requests | `WHERE wantsGuidance=true` → list `whatIWant` |
| caseComplexity distribution | `GROUP BY computed.caseComplexity` |
| Score over time | `GROUP BY iteration, AVG(aiNativeScore)` |
