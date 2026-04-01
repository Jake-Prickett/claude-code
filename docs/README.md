# Claude Code Architecture — Documentation

This folder documents the key architectural patterns observed in Claude Code's source, extracted for reference when designing similar AI agent systems.

## Documents

| Document | Topic |
|----------|-------|
| [multi-agent-orchestration.md](./multi-agent-orchestration.md) | Coordinator/worker pattern, fork vs. spawn, task lifecycle, agent communication |
| [prompt-architecture.md](./prompt-architecture.md) | System prompt design, modular sections, tool prompt conventions |
| [memory-system.md](./memory-system.md) | Persistent memory, the Dream consolidation engine, memory file layout |
| [permission-system.md](./permission-system.md) | Permission modes, risk classification, tool execution safety |

## Quick-Reference: Concepts That Make It Work

| Concept | Where it lives | Why it matters |
|---------|---------------|----------------|
| Coordinator/worker split | `coordinator/coordinatorMode.ts` | Coordinator has a different identity — it only orchestrates, never does hands-on work |
| task-notification XML | `coordinatorMode.ts` | Workers return results as user-role XML messages — keeps conversation state clean, enables async |
| Shared scratchpad | `coordinatorMode.ts`, `utils/permissions/filesystem.ts` | Cross-worker durable state without message-passing overhead |
| Fork vs. spawn | `tools/AgentTool/prompt.ts` | Context-window optimization — reuse loaded context or start clean based on overlap |
| Modular system prompt | `constants/systemPromptSections.ts`, `constants/prompts.ts` | Static sections cache globally; dynamic sections recompute per-session |
| SYSTEM_PROMPT_DYNAMIC_BOUNDARY | `constants/prompts.ts` | Separates globally-cacheable content from per-session content — significant cost savings at scale |
| Synthesis mandate | `coordinatorMode.ts` | Coordinator must prove it understood findings before issuing follow-up prompts |
| DANGEROUS_uncachedSystemPromptSection | `constants/systemPromptSections.ts` | Named convention forcing documentation of why a section busts the prompt cache |
| Memory index vs. topic files | `memdir/memdir.ts` | Index always loaded (≤200 lines); topic files loaded selectively by relevance |
| Dream consolidation gates | `services/autoDream/` | 24h + 5 sessions + lock — prevents both over-consolidation and staleness |
| YOLO classifier | `utils/permissions/yoloClassifier.ts` | Fast side-query to ML model for per-call auto-approval — separate from main conversation |
| Feature flags (`feature()`) | Throughout | Compile-time dead-code elimination — internal-only features never ship to external builds |
| Undercover mode | `utils/undercover.ts` | Prevents internal codenames from leaking in commits/PRs when in public repos |
| Denial tracking | `utils/permissions/denialTracking.ts` | User denials surface back to the model so it changes approach rather than retrying |

## Key Design Principles

These cross-cut all four documents and are worth keeping in mind:

1. **Prompts are code.** Every tool, mode, and agent has a dedicated `prompt.ts` file. Prompt text is versioned, tested, and reviewed like source code.
2. **Context is a resource.** Every design decision—fork vs. spawn, prompt caching, memory pruning—is ultimately about managing what fits in the model's context window at a given cost.
3. **Synthesis is the coordinator's job.** A coordinator that delegates understanding to workers produces shallow results. The coordinator must read, comprehend, and produce specific specs.
4. **Parallelism is a first-class primitive.** The agent tool prompt explicitly calls parallelism a "superpower." Independent work fans out; write-heavy work serializes.
5. **Workers can't see your conversation.** Every worker prompt must be fully self-contained.
