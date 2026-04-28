# AI-Пульс — опросник об AI-нативности команды

Внутренний опросник Rezolve AI: какие AI-инструменты используются, насколько глубоко, где боли и что нужно. Задача — получить реальную картину и принять конкретные решения: подписки, помощь с задачами, накопление практик внутри команды.

Детальный план вопросов и решений: [PLAN.md](./PLAN.md)

---

## Текущее состояние

Сейчас в репозитории — **статический HTML-прототип** (`index.html`). Полностью функциональный интерфейс: все разделы, матрица инструментов, голосовой ввод, drag & drop, валидация, архетип-карточка. Бэкенда нет — данные никуда не отправляются.

Следующий шаг: перевод на Next.js + подключение MongoDB и Claude API согласно структуре ниже.

---

## Стек

| Слой | Сейчас (прототип) | Production |
|---|---|---|
| Frontend | Статический `index.html` | Next.js 15 + TypeScript |
| Голосовой ввод | Web Speech API | Web Speech API |
| Файлы / скриншоты | Нативный upload + drag & drop | То же |
| AI-обработка | Mock (js) | Claude API (`claude-sonnet-4-6`) |
| База данных | — | MongoDB |
| Reporting | — | Notion (корп. workspace Rezolve Ai) |
| Деплой | GitHub Pages | Vercel или внутренний сервер |

---

## Дизайн-система

Тёмная тема. Шрифты: **Manrope** (заголовки), **Inter Tight** (текст).

```html
<link href="https://fonts.googleapis.com/css2?family=Manrope:wght@400;700;800&family=Inter+Tight:wght@400;500;600;700&display=swap" rel="stylesheet">
```

### CSS-переменные (тёмная тема)

```css
:root {
  --blue:   #478BC9;
  --bg:     #080f1f;
  --card:   rgba(255,255,255,0.05);
  --card2:  rgba(255,255,255,0.08);
  --border: rgba(255,255,255,0.1);
  --text:   rgba(255,255,255,0.9);
  --sub:    rgba(255,255,255,0.6);
  --muted:  rgba(255,255,255,0.35);
}
```

### Лого

```
https://rezolve.com/wp-content/uploads/2025/07/rezolve-logo-12-2025.svg
```

---

## Структура файлов

```
ai-pulse-survey/
├── index.html               # Прототип — всё в одном файле
├── prompts/
│   └── v1/                  # Промпты зафиксированы — не менять между итерациями
│       ├── archetype.md     # Генерация архетип-карточки (синхронно)
│       ├── classify-case.md
│       ├── classify-pain.md
│       ├── classify-skills.md
│       └── normalize-tool.md
├── PLAN.md
├── README.md
└── archive-conversation.md  # История решений по дизайну опросника
```

### Production-структура (Next.js, будущее)

```
├── app/
│   ├── page.tsx
│   ├── survey/page.tsx
│   ├── done/page.tsx
│   └── api/
│       ├── submit/route.ts
│       ├── summarize/route.ts
│       ├── classify/route.ts
│       └── tools/route.ts
├── components/
│   ├── sections/
│   ├── DinoProgress.tsx
│   ├── ArchetypeCard.tsx
│   └── VoiceInput.tsx
├── lib/
│   ├── mongodb.ts
│   ├── notion.ts
│   ├── claude.ts
│   └── scoring.ts
└── types/survey.ts
```

---

## База данных — MongoDB

### Коллекция `responses`

```ts
{
  _id: ObjectId,
  createdAt: Date,
  updatedAt: Date,

  name: string,
  whoFilled: "self" | "ai-help-write" | "ai-cases" | "ai-filled-checked" | "agent",

  // Разделы 2–5
  taskCategories: string[],
  taskOther: string | null,
  usesSavedPrompts: boolean,
  savedPromptsText: string | null,
  usesApiIntegrations: boolean,
  sharingPreference: "yes" | "anon" | "no",
  frictionScore: 1 | 2 | 3 | 4 | 5,
  blockersTechnical: string[],
  blockersHuman: string[],
  blockersOther: string | null,

  // Открытые поля
  topPrompts: string | null,
  workCase: string | null,
  workCaseFileUrl: string | null,
  personalCase: string | null,
  whatFailed: string | null,
  whatIWant: string | null,
  comments: string | null,
  metaAnswer: string | null,
  usageSnapshotUrl: string | null,

  // Вычисленные поля (Claude API, async)
  computed: {
    classifiedAt: Date | null,
    archetype: "explorer" | "practitioner" | "integrator" | "ambassador" | null,
    aiNativeScore: number | null,   // 0–100, только для аналитики
    summaryCard: string | null,
    caseType: string | null,
    caseComplexity: "rote" | "augmentation" | "transformation" | null,
    painType: string | null,
    skillsDomains: string[],
  },

  metrics: {
    stackBreadthWork: number,
    useCaseDiversity: number,
    depthScore: number,        // 0–10
    sharingPropensity: number, // 0–3
    demandSignal: number,      // 0–2
    personalInvestment: number,
    droppedToolsCount: number,
  }
}
```

### Коллекция `tool_selections`

```ts
{
  _id: ObjectId,
  createdAt: Date,
  responseId: ObjectId,
  toolId: ObjectId,
  toolName: string,
  status: "use" | "dropped" | "no" | "try",
  paymentType: "company" | "self" | "free" | "want-coverage" | null,
  frequency: string | null,
  accessMethods: string[],
  wantCoverage: boolean,
  usageIntent: string | null,   // для статуса "try"
  dropReason: string | null,
}
```

### Коллекция `tools`

```ts
{
  _id: ObjectId,
  canonicalName: string,
  category: "chat" | "coding" | "design" | "voice" | "data" | "other",
  inBaseList: boolean,
  approved: boolean,            // показывать следующим при 2+ упоминаниях
  mentionCount: number,
  rawInputs: string[],
}
```

---

## Пайплайн обработки

```
[Submit]
    │
    ▼
1. scoring.ts — рассчитать метрики синхронно
    │
    ▼
2. MongoDB: сохранить responses + tool_selections
    │
    ├─► [Синхронно] Claude API → archetype.md
    │       → показать архетип-карточку
    │
    └─► [Асинхронно]
            classify-case.md  → computed.caseType + caseComplexity
            classify-pain.md  → computed.painType
            classify-skills.md → computed.skillsDomains
            normalize-tool.md → обновить tools
            → MongoDB: update computed + updatedAt
            → Notion: sync (если доступ получен)
```

---

## Метрики и скоринг

### Компоненты AI-Native Score (0–100)

| Метрика | Вес | Источник |
|---|---|---|
| Frequency Score (0–4) | 25% | Матрица инструментов, Раздел 1 |
| Depth Score (0–10) | 20% | Раздел 3 |
| Use Case Diversity (0–4) | 15% | Раздел 2 |
| Stack Breadth | 10% | Раздел 1 |
| Sharing Propensity (0–3) | 5% | Раздел 4 |
| Demand Signal (0–2) | 5% | Раздел 5 |

> AI-Native Score хранится только в аналитике. Респонденту показывается только архетип.

### Depth Score (0–10)

```
usesSavedPrompts ? 2 : 0
+ min(savedPromptsCount, 5) * 0.4
+ usesApiIntegrations ? 3 : 0
+ usesAiInEditor ? 3 : 0
```

### Архетипы

| Архетип | Score | Суть |
|---|---|---|
| 🦕 Исследователь | 0–25 | Редко, поверхностно, без системы |
| 🚀 Практик | 26–55 | Регулярно, 1–2 инструмента в фокусе |
| ⚡ Интегратор | 56–80 | Широкий стек, автоматизации, меняет процессы |
| 🧠 Амбассадор | 81–100 | Всё выше + делится с командой |

---

## Промпты-классификаторы

Хранятся в `prompts/v1/`. **Не изменяются** — иначе сравнение между итерациями невалидно. При необходимости: создать `prompts/v2/`, зафиксировать в документации.

| Файл | Назначение | Вызов |
|---|---|---|
| `archetype.md` | Архетип-карточка для респондента | Синхронно |
| `classify-case.md` | Тип + сложность кейса | Async |
| `classify-pain.md` | Тип боли | Async |
| `classify-skills.md` | Домены промптов | Async |
| `normalize-tool.md` | Нормализация инструмента из «другое» | Async |

---

## Переменные окружения

```env
MONGODB_URI=
MONGODB_DB=ai_pulse

ANTHROPIC_API_KEY=

NOTION_API_KEY=
NOTION_DB_RESPONSES=
```

---

## Открытые вопросы

- [ ] Запросить доступ к корпоративному Notion workspace Rezolve Ai
- [ ] Production-деплой: Vercel или внутренний сервер?
- [ ] Проверка политики безопасности на внешние API (MongoDB Atlas, Claude)
- [ ] Доступ к `/admin` аналитике: basic auth, Okta или magic link?
