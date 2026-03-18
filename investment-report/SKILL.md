---
name: investment-report
description: >
  Generates investment analysis reports by reading new articles from sponsr.ru/crimsonanalytics
  (CrimsonAlter blog), cross-referencing with a local knowledge base of investment lectures,
  and producing a professional DOCX report in conversational Russian.
  Use this skill whenever the user mentions CrimsonAlter, sponsr.ru, crimsonanalytics,
  "read new articles", "investment report", "инвестиционные идеи", or asks to analyze
  financial/political articles and generate investment insights. Also trigger when the user
  asks to "update the report", "check for new articles", or references the current_situation
  or knowledge_base folders.
---

# CrimsonAlter Investment Report Generator

Generate investment analysis reports from CrimsonAlter Analytics articles on sponsr.ru, written in the style of Alexey Antonov's investment lectures.

## Prerequisites

- **Chrome MCP tools** (Claude in Chrome) — needed to browse sponsr.ru and read articles
- **docx-js** (`npm install -g docx`) — for DOCX generation
- **User's investment folder** must contain:
  - `knowledge_base/` — lecture transcripts (style and domain reference)
  - `current_situation/` — output folder for reports

## Workflow Overview

The process has 6 stages. Execute them in order.

### Stage 1: Read the Style Guide

Before doing anything else, read `references/style_guide.md` from this skill's directory. It contains the distilled writing style and a topic-to-file mapping for the knowledge base. This is your tone compass for the entire report.

### Stage 2: Review Previous Reports

Before browsing for new articles, check what already exists in `current_situation/`. This step gives you context continuity — you'll know what's already been analyzed and can build on it rather than starting from scratch.

1. List files in `current_situation/` and identify the most recent 1-2 `.docx` reports by filename date
2. Convert the most recent report to readable text using pandoc:
   ```bash
   pandoc <most_recent_report>.docx -o /tmp/previous_report.md
   ```
3. Read the converted markdown to extract:
   - **Date range covered** — which articles were analyzed last time
   - **Key investment theses** — what positions/ideas were recommended
   - **Ongoing narratives** — situations that were developing (e.g., rate cuts, sanctions normalization, Iran conflict)
   - **Specific companies/sectors** discussed and the stance taken on them
   - **Timeline items** — what events were flagged as upcoming catalysts

This context serves three purposes in the new report:
- **Continuity**: reference previous analysis naturally ("как мы обсуждали в предыдущем обзоре...", "в прошлый раз мы отмечали, что...")
- **Evolution tracking**: show how situations developed ("тогда ЦБ только намекал на снижение ставки — теперь сигналы стали явными")
- **Thesis updates**: confirm, revise, or retire previous investment ideas ("тезис по Полюсу остаётся в силе", "ситуация с застройщиками ухудшилась — наш прогноз подтвердился")

If no previous reports exist, skip this stage — it's the first report.

### Stage 3: Browse for New Articles

1. Get Chrome tab context (`tabs_context_mcp` with `createIfEmpty: true`)
2. Navigate to `https://sponsr.ru/crimsonanalytics/`
3. Wait 3 seconds for page load
4. Read the page to identify articles and their dates
5. Determine which articles are new — use the date range from Stage 2 to know the cutoff. If no previous report exists, take the last 3 articles

**Article tiers on sponsr.ru:**
- "Новостной деск" (Пропуск на новостной деск) — paid tier, usually accessible if logged in
- "Кофе с банкиром" — higher paid tier, may be locked
- "Бесплатный" — free, always accessible
- "[Заметки]" — short free notes

Focus on accessible articles. If an article is locked behind a paywall you can't access, skip it and note this in the report.

### Stage 4: Read Each Article

For each new article:

1. Click on the article link or navigate to its URL
2. Wait 3 seconds for load
3. Use `read_page` with `depth: 3` and `max_chars: 80000` to extract content
4. If `get_page_text` works (article under 50k chars), that's also fine
5. Extract key insights, arguments, investment implications, and notable quotes

**What to extract from each article:**
- Main thesis / argument
- Specific companies, sectors, or assets mentioned
- Political/economic signals and their investment implications
- Actionable ideas (explicit or implied)
- Memorable phrases or frameworks that resonate with the knowledge base style

### Stage 5: Cross-Reference with Knowledge Base

Based on the topics covered in the articles, read 2-3 relevant knowledge base files (first 200 lines each). The topic-to-file mapping is in `references/style_guide.md`.

The purpose is NOT to summarize the knowledge base, but to:
- Borrow frameworks and mental models (e.g., "risk transfer", "tilt", "old money mentality")
- Find concepts that connect to the articles' insights
- Adopt the conversational, practical tone

### Stage 6: Generate the DOCX Report

**Critical rule**: Write the build script (`.js` file) in the session's temp directory (`/sessions/*/`), NOT in the user's output folder. Only the final `.docx` goes into `current_situation/`.

#### File naming
`investment_ideas_<DDmon>_<YYYY>.docx` — e.g., `investment_ideas_18mar_2026.docx`

#### Document structure

Use `docx-js` to generate the report. The DOCX skill should be consulted for technical details on how to use docx-js correctly (styles, numbering, tables, etc.).

**Required sections:**

1. **Title page** — document title, subtitle reflecting main themes, date, source attribution, disclaimer in red italic
2. **Overview** — "Что произошло" — big picture in 2-3 paragraphs. If a previous report exists, open with how the situation evolved since then (not a summary of the old report, but a bridge: "С момента нашего последнего обзора [date] произошло X, Y, Z")
3. **Thematic sections** — one per major topic from the articles. Each should have:
   - Clear heading describing the topic
   - Analysis mixing CrimsonAlter's insights with knowledge base frameworks
   - Bold text for key takeaways
   - "Для инвестора:" or "Вывод:" connecting to practical action
4. **Concrete investment ideas** — numbered list of specific opportunities with strategy notes. For each idea, note if it's new or carried from a previous report (and whether the thesis strengthened/weakened)
5. **What to avoid** — specific pitfalls relevant to current situation
6. **Portfolio principles** — timeless wisdom combining both sources
7. **Timeline** — upcoming events/catalysts to watch
8. **Conclusion** — end with optimism and the signature sign-off style

#### DOCX formatting specs

```javascript
// Page: A4, 1-inch margins
page: { size: { width: 11906, height: 16838 }, margin: { top: 1440, right: 1440, bottom: 1440, left: 1440 } }

// Styles: Arial throughout
// Heading 1: 18pt bold, color #1B4F72
// Heading 2: 14pt bold, color #2E75B6
// Body: 12pt Arial

// Header: italic gray text with blue bottom border
// Footer: centered page number with gray top border

// Lists: use LevelFormat.BULLET or LevelFormat.DECIMAL with proper numbering config
// NEVER use unicode bullets manually

// Disclaimer: 9pt gray text with top border separator
```

#### After generation

1. Run `python scripts/office/validate.py <output.docx>` to validate
2. Delete the build script from temp directory
3. Provide the user a `computer://` link to the file

### Writing the Report Content

The report is written entirely in Russian. Here's the voice to aim for:

**DO:**
- Explain complex geopolitical/financial situations as if talking to a smart friend
- Use bold for key insights within flowing paragraphs
- Connect every observation to "what does this mean for my money?"
- Reference Antonov's frameworks naturally ("как говорит Антонов...", "вспомним принцип...")
- Use CrimsonAlter's signature phrases where they fit ("хрен его знает", "мурчите громче")
- Acknowledge uncertainty honestly — forced confidence is worse than honest doubt
- **Reference previous reports** — show how the narrative evolved: "в прошлом обзоре мы отмечали X — теперь Y", "тезис по Z остаётся в силе / подтвердился / требует пересмотра"
- **Update investment ideas** — explicitly state whether previous ideas are still valid, strengthened, or should be retired
- End with a calming, slightly humorous sign-off

**DON'T:**
- Write like a bank report — no dry corporate language
- Make bullet points without substance — each point is a full thought
- Panic or amplify fear — the whole point is rational analysis
- Forget the disclaimer: "не является инвестиционной рекомендацией"
- Put the .js build file in the user's output folder

## Error Handling

- **Page won't load**: Wait longer, try refreshing. If still fails, get fresh tab context.
- **Article behind paywall**: Note it in the report, work with what's accessible.
- **get_page_text too large**: Use `read_page` with `depth: 3, max_chars: 80000` instead.
- **docx module not found at standard path**: Search with `find / -name "docx" -type d` and use the full path in `require()`.
- **Validation fails**: Read the error, fix the JS, regenerate.
