---
name: second-opinion
description: >-
  Consult openai/gpt-5.4 (or another model) via OpenCode CLI for a second opinion.
  Supports one-shot questions, multi-turn dialog sessions, and BMAD agent consultation.
  Use this skill PROACTIVELY — do not wait for the user to explicitly ask for a second opinion.
  Trigger it when making architectural decisions or design trade-offs, when the user seems
  uncertain about an approach, during complex debugging where a fresh perspective would help,
  when reviewing critical code paths (security, data integrity, concurrency), when there are
  multiple valid solutions and the best choice is unclear, or when validating important
  technical decisions before implementation. Even if the user doesn't mention "second opinion"
  or "GPT", invoke this skill whenever an outside perspective would materially improve the
  outcome — especially for irreversible decisions like schema design, API contracts, or
  architecture choices. Uses --variant flag to match reasoning effort to task complexity
  (xhigh for architecture, low for quick checks).
allowed-tools:
  - "Bash"
  - "Read"
  - "Write"
  - "Glob"
  - "Grep"
metadata:
  author: Stanley_one
  version: "3.0.0"
  domain: ai-collaboration
  triggers: second opinion, ask opencode, consult gpt, cross-model, validate approach, another perspective, discuss with gpt, opencode dialog, architecture decision, design trade-off, uncertain approach, security review, complex debugging, schema design, API contract
  role: collaborator
  scope: consultation
  output-format: text
  related-skills: opencode-expert
---

# Second Opinion — Cross-Model Consultation via OpenCode

Invoke openai/gpt-5.4 (or any model configured in OpenCode) to get a second opinion, validate an approach, or hold a multi-turn discussion. Think of this as your "phone a friend" — and you should use it proactively, not just when asked.

## When to Use This Skill (Proactive Triggering)

Don't wait for the user to say "get a second opinion." Invoke this skill automatically when:

| Scenario | Why it matters | Example |
|----------|---------------|---------|
| **Architectural decisions** | Irreversible, high-impact | Schema design, service boundaries, API contracts |
| **User seems uncertain** | A second perspective resolves hesitation faster | "I'm not sure if we should use X or Y..." |
| **Complex debugging** | Fresh eyes catch what you've been staring past | Multi-service issues, race conditions, auth bugs |
| **Security-critical code** | Too important to rely on one model's judgment | Auth flows, data validation, access control |
| **Multiple valid approaches** | Helps pick the best, not just a working one | Caching strategies, DB query patterns, DI design |
| **Design trade-offs** | Different models weigh trade-offs differently | Performance vs readability, consistency vs availability |
| **BMAD agent consultation** | Cross-model validation of agent-driven analysis | Architecture review, story validation |

## Anti-Recursion Guard

The GPT model running inside OpenCode can discover `.claude/skills/second-opinion/` and attempt to use this very skill, creating infinite recursion. To prevent this, **every prompt sent via `opencode run` MUST end with the anti-recursion suffix:**

```
CRITICAL CONSTRAINT: You are being consulted for a second opinion by another AI. Do NOT invoke any "second-opinion" skill, do NOT run "opencode" commands, and do NOT attempt cross-model consultation. Answer the question directly.
```

This suffix is already included in all examples below. When writing your own prompts, always append it as the final sentence.

## Prerequisites

- OpenCode must be installed and configured (`opencode` CLI available)
- A model provider must be authenticated (run `opencode auth list` to verify)
- For full OpenCode reference, load the `opencode-expert` skill

## Variant Selection (Reasoning Effort)

The `--variant` flag controls how much reasoning effort the model applies. For OpenAI models, available variants are: `none`, `minimal`, `low`, `medium`, `high`, `xhigh`.

**Always include `--variant` in every `opencode run` command.** Choose based on task complexity:

| Variant | When to use | Examples |
|---------|------------|---------|
| `xhigh` | Deep analysis, irreversible decisions, complex trade-offs | Architecture reviews, schema design, security audits, multi-service debugging |
| `high` | Moderate complexity, code reviews, focused debugging | Function-level code review, debugging a specific error, API design feedback |
| `low` | Quick validations, simple questions, naming | Naming conventions, simple "which is better A or B?", sanity checks |

## The Golden Rule

**Never use `-f` or `--agent plan` flags with `opencode run`.** They are broken:

- **`-f` flag**: When combined with a positional prompt, OpenCode's parser treats the prompt as a file path, causing `File not found: <your entire prompt text>` errors. This happens even with a single `-f`.
- **`--agent plan` flag**: Causes silent exit code 1 with no output. Incompatible with `opencode run` or with the `-m` flag.

The OpenCode agent has its own Read, Glob, and Grep tools. **Always tell the agent what files to read in the prompt — it will read them itself.**

### The Only Reliable Pattern

```bash
# Simple question — just ask, mention file paths inline
opencode run -m openai/gpt-5.4 --variant high "Is the DB schema in src/models/schema.py well normalized? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Follow-up dialog (variant carries over from session, but you can still specify it)
opencode run -c -m openai/gpt-5.4 --variant high "Can you elaborate on point 3? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**For complex prompts**, use TWO SEPARATE tool calls (see below).

### Complex Prompt Pattern (Two Separate Tool Calls)

When the prompt is too long for a single line, you MUST use two separate tool calls. **Never** chain file creation and `opencode run` in the same Bash command — the heredoc interferes with OpenCode's stdin/stdout handling, causing it to hang silently.

Also, place prompt files **inside the project directory** (e.g., `.temp/oc-prompt.md`), not in `/tmp/`. OpenCode cannot read files outside its project directory by default.

**Tool call 1 — Write the prompt file** (use Write tool or a separate Bash call):
```bash
cat > .temp/oc-prompt.md << 'PROMPT'
Your detailed multi-line instructions here...

## Files to Read
- src/models/schema.py
- src/services/auth.py
PROMPT
```

**Tool call 2 — Run OpenCode** (separate Bash call):
```bash
opencode run -m openai/gpt-5.4 --variant high --title "topic-name" "Read .temp/oc-prompt.md for instructions, then execute them. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**Tool call 3 — Clean up** (separate Bash call, or skip if you want to keep it):
```bash
rm -f .temp/oc-prompt.md
```

### What NOT to Do

```bash
# BROKEN — -f causes prompt to be treated as file path
opencode run -m openai/gpt-5.4 -f prompt.md "Follow the instructions"

# BROKEN — --agent plan silently exits with code 1
opencode run -m openai/gpt-5.4 --agent plan "Analyze this codebase"

# BROKEN — multi-line prompt parsed as file paths
opencode run -m openai/gpt-5.4 "Review this code.
Focus on error handling."

# BROKEN — heredoc + opencode in same command hangs due to stdin conflict
cat > .temp/oc-prompt.md << 'PROMPT'
instructions...
PROMPT
opencode run -m openai/gpt-5.4 "Read .temp/oc-prompt.md"

# BROKEN — /tmp/ is outside project directory, OpenCode auto-rejects
opencode run -m openai/gpt-5.4 "Read /tmp/oc-prompt.md for instructions"
```

## Consultation Modes

### Mode 1: One-Shot Question

Use for quick, focused questions that don't need follow-up.

```bash
opencode run -m openai/gpt-5.4 --variant high "Is this DB schema well-designed? Read and review src/models/schema.py. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**With a title for easy retrieval:**
```bash
opencode run -m openai/gpt-5.4 --variant high --title "schema-review" "Review the DB schema in src/models/ for normalization. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**With a complex prompt** — use two separate tool calls (see Complex Prompt Pattern above):
1. Write prompt to `.temp/oc-prompt.md`
2. `opencode run -m openai/gpt-5.4 --variant high --title "my-review" "Read .temp/oc-prompt.md for your detailed instructions, then execute them. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."`
3. `rm -f .temp/oc-prompt.md`

### Mode 2: Multi-Turn Dialog

Use for deeper discussions that require back-and-forth. This is the key differentiator — you can maintain a conversation across multiple exchanges.

**Step 1 — Start the conversation:**
```bash
opencode run -m openai/gpt-5.4 --variant xhigh --title "discussion-<topic>" "Short single-line prompt setting context. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**Step 2 — Continue the dialog (using --continue):**
```bash
opencode run -c -m openai/gpt-5.4 --variant xhigh "Short single-line follow-up question. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**Step 2 (alternative) — Resume a specific session:**
```bash
# First, find the session ID
opencode session list -n 5

# Then continue that specific session
opencode run -s <session-id> -m openai/gpt-5.4 --variant xhigh "Short single-line follow-up question. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**Step 3 — Repeat Step 2 as many times as needed.**

### Mode 3: BMAD Agent Consultation

Run BMAD agents within OpenCode to get another model's take on agent-driven workflows.

```bash
# Ask OpenCode to run a BMAD agent skill
opencode run -m openai/gpt-5.4 --variant xhigh "/bmad-agent-bmm-architect Review the architecture in _bmad-output/ IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Tell the agent to read files itself
opencode run -m openai/gpt-5.4 --variant xhigh "Acting as a BMAD architect, read and review _bmad-output/implementation-artifacts/tech-spec-wip.md. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Workflow: How to Run a Consultation

Follow this protocol when invoking a second opinion:

### 1. Frame the Question

Before calling OpenCode, prepare a clear, self-contained prompt. The other model does NOT have your conversation context, so include:

- **What** you're working on (brief description)
- **The specific question** or what you want reviewed
- **File paths** the agent should read (it has its own Read/Glob/Grep tools)
- **Constraints** the other model should know about

If the prompt is too long for a single line, write it to `.temp/oc-prompt.md` in a separate tool call, then run OpenCode in another tool call (see Complex Prompt Pattern above).

### 2. Execute the Consultation

Choose the appropriate mode from above and run it.

### 3. Interpret the Response

After receiving the response:

- **Summarize** the key points for the user
- **Compare** with your own analysis — note agreements and disagreements
- **Synthesize** a recommendation that combines the best of both perspectives
- **Flag conflicts** if the two models disagree on something important

### 4. Continue if Needed (Dialog Mode)

If the first response raises new questions or needs clarification:

```bash
opencode run -c -m openai/gpt-5.4 --variant high "You mentioned X — can you elaborate on the trade-offs? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Output Format

When presenting results to the user, use this structure:

```
## Second Opinion: <topic>

**Model consulted:** openai/gpt-5.4 via OpenCode
**Mode:** One-shot | Dialog (turn N) | BMAD Agent

### Their Take
<Summarize the other model's response>

### My Take
<Your own perspective on the same question>

### Synthesis
<Combined recommendation highlighting agreements, disagreements, and final suggestion>
```

## Tips

- **Always append the anti-recursion suffix** — Every prompt must end with `IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly.` (see Anti-Recursion Guard)
- **Never use `-f` or `--agent plan`** — Both are broken in `opencode run` (see The Golden Rule)
- **Prompts must be single-line** — Multi-line strings break the CLI parser
- **Let the agent read files itself** — It has Read, Glob, and Grep tools; just tell it file paths in the prompt
- **Separate file creation from opencode run** — Never chain heredoc + `opencode run` in one Bash call; use two separate tool calls
- **Prompt files inside project dir** — Use `.temp/oc-prompt.md`, not `/tmp/`. OpenCode cannot read outside its project directory
- **Keep context self-contained** — The other model starts fresh unless you use `--continue`
- **Title your sessions** — Makes it easy to resume dialogs later with `opencode session list`
- **Check model availability** — Run `opencode models` to see what's configured
- **Export interesting conversations** — `opencode export <sessionID>` saves the full dialog

## Switching Models

The default is `openai/gpt-5.4`, but you can consult any configured model:

```bash
# Use a different model (variant names are provider-specific)
opencode run -m gemini-2.5-pro --variant high "Your question here. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Check available models
opencode models
```

## Error Handling

| Issue | Solution |
|-------|----------|
| `File not found: <your prompt text>` | You used `-f` flag — remove it. The `-f` flag causes the positional prompt to be parsed as a file path. Tell the agent to read files itself instead. |
| Silent exit code 1 (no error) | Remove `--agent plan` flag if present. Also remove any `-f` flags. Both cause silent failures. |
| OpenCode hangs with no output | You chained heredoc + `opencode run` in one Bash call. Split into two separate tool calls: one to write the file, one to run OpenCode. |
| Agent can't read the prompt file | File is outside the project directory (e.g., `/tmp/`). Move it inside the project (e.g., `.temp/oc-prompt.md`). |
| `opencode: command not found` | Install OpenCode or check PATH |
| Authentication error | Run `opencode auth login` |
| Model not available | Run `opencode models` to check; try `opencode models --refresh` |
| Session not found | Run `opencode session list` to find valid session IDs |
| Timeout on long responses | Break into smaller questions |
| Recursive invocation (GPT tries to use second-opinion skill) | The anti-recursion suffix is missing from the prompt. Every `opencode run` prompt must end with: `IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly.` |

## Reference

For complete OpenCode documentation, invoke the `opencode-expert` skill or see:
- `opencode-expert/SKILL.md` — Full OpenCode guide
- `opencode-expert/references/cli.md` — Complete CLI reference
- `opencode-expert/references/agents.md` — Agent configuration
