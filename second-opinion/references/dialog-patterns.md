# Dialog Patterns for Multi-Turn Consultations

> **The Golden Rule:** Never use `-f` or `--agent plan` flags. Keep prompts single-line.
> The agent has Read/Glob/Grep tools — always tell it what files to read in the prompt.
> **Never chain heredoc + opencode run in one Bash call** — use separate tool calls.
> **Prompt files must be inside the project directory** (e.g., `.claude/oc-prompt.md`), not `/tmp/`.
> **Anti-recursion:** Every prompt MUST end with: `IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly.`

## Pattern 1: Architecture Review Dialog

```bash
# Turn 1 — Set the stage (agent reads file itself) → xhigh for architecture
opencode run -m openai/gpt-5.4 --variant xhigh --title "arch-review" \
  "Review docs/architecture.md for scalability concerns in our multi-tenant FastAPI + EdgeDB setup. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 2 — Dig deeper
opencode run -c -m openai/gpt-5.4 --variant xhigh \
  "Good point about connection pooling. How would you handle tenant isolation at the DB level? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 3 — Challenge
opencode run -c -m openai/gpt-5.4 --variant xhigh \
  "Row-level security vs separate schemas — what are the operational trade-offs at 1000+ tenants? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Pattern 2: Debugging Session

```bash
# Turn 1 — Describe the problem, let agent read the file → high for debugging
opencode run -m openai/gpt-5.4 --variant high --title "debug-auth" \
  "Users get 403 after token refresh. Refresh returns 200 with new token but subsequent requests fail. Read src/middleware/auth.py. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 2 — Point to more context
opencode run -c -m openai/gpt-5.4 --variant high \
  "Now also read src/services/auth_service.py for the token refresh logic. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 3 — Test hypothesis
opencode run -c -m openai/gpt-5.4 --variant high \
  "Could the old token be cached in the API gateway? Check config/gateway.yaml. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Pattern 3: Code Review Debate

```bash
# Turn 1 — Tell agent to read and review (NO -f flag) → high for code review
opencode run -m openai/gpt-5.4 --variant high --title "review-rag" \
  "Read src/services/rag_pipeline.py and review it for chunking strategy, retrieval quality, and error handling. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 2 — Disagree and discuss
opencode run -c -m openai/gpt-5.4 --variant high \
  "I disagree about fixed-size chunks. Our docs have clear section boundaries. Wouldn't semantic chunking by headings be better? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Turn 3 — Ask for implementation plan
opencode run -c -m openai/gpt-5.4 --variant high \
  "Outline the ideal chunking approach for markdown docs with H1-H3 structure and code blocks. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Pattern 4: BMAD Agent Cross-Consultation

```bash
# Get architect perspective from GPT (let it read the file) → xhigh for architecture
opencode run -m openai/gpt-5.4 --variant xhigh --title "bmad-arch" \
  "You are a senior software architect. Read _bmad-output/implementation-artifacts/tech-spec-wip.md and identify gaps, risks, and missing NFRs. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Then compare with BMAD architect locally
# (invoke /bmad-agent-bmm-architect in Claude for comparison)
```

## Pattern 5: Quick Validation Chain

For rapid-fire validations that don't need multi-turn — use `low` variant:

```bash
# Validate naming → low for simple question
opencode run -m openai/gpt-5.4 --variant low "BotSessionManager vs ConversationOrchestrator — which is better for a chat session manager across multiple bot platforms? IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Validate approach → high since it involves trade-offs
opencode run -m openai/gpt-5.4 --variant high "Webhook retry: exponential backoff with jitter in-app vs dead-letter queue with scheduled reprocessing? Context: FastAPI + Celery + Redis. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."

# Validate query (agent reads the file itself) → low for quick check
opencode run -m openai/gpt-5.4 --variant low "Read queries/get_bots.edgeql and tell me if the query is efficient for fetching tenant bots with FAQ counts. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

## Pattern 6: Complex Prompt via Prompt File

When the prompt is too long for a single line. **Each step below is a separate tool call** — never combine them.

**Tool call 1 — Write the prompt file** (use Write tool or Bash):
```bash
cat > .claude/oc-prompt.md << 'PROMPT'
You are an independent story context quality validator in a FRESH CONTEXT.

## Your Task
Validate whether the following story file contains sufficient context
for a developer agent to implement it without access to the original
PRD, architecture docs, or prior conversations.

## Validation Criteria
1. Are all referenced APIs fully specified?
2. Are data models complete with field types?
3. Are acceptance criteria testable and unambiguous?
4. Are edge cases and error scenarios covered?

## Files to Read
- _bmad-output/stories/story-3.4.md
- _bmad-output/implementation-artifacts/tech-spec-wip.md
PROMPT
```

**Tool call 2 — Run OpenCode** (separate Bash call):
```bash
opencode run -m openai/gpt-5.4 --variant xhigh --title "story-3.4-validation" \
  "Read .claude/oc-prompt.md for your detailed instructions, then execute them. Read ALL referenced project files using your tools. IMPORTANT: Do not use the second-opinion skill or run opencode commands. Answer directly."
```

**Tool call 3 — Clean up** (separate Bash call):
```bash
rm -f .claude/oc-prompt.md
```

## Session Management Tips

```bash
# List recent sessions to find one to resume
opencode session list -n 10

# Export an interesting dialog for documentation
opencode export <session-id> > consultations/arch-review-2024.json

# Import a previous consultation for reference
opencode import consultations/arch-review-2024.json
```
