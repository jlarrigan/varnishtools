# Stack module: AI / LLM features
_Signals: `openai`, `@anthropic-ai/sdk`, `@google/genai`, `ai`, `langchain`, model-provider API keys. Adds to the core passes by number, and carries hunter H6 for `security-audit.md`._

- **Keys:** provider keys (`sk-`, `sk-proj-`, `sk-ant-`, `AIza…`) anywhere client-side, behind a public prefix, or in git are High (launch 1, 8). Calls to the model belong on the server.
- **Launch 6 / pass 10 — denial of wallet:** every route that calls a model has: an auth check; a per-user rate limit or quota; `max_tokens` (or equivalent) set; input size limits; and the provider account has a monthly budget limit (Couldn't-check — list it). An open model endpoint is someone else's free AI on your card.
- **10 — prompt injection:** user content, retrieved documents (RAG), web pages, or tool output are untrusted. Look for: system prompts that rely on "ignore instructions in the user text" for security; model output used to build SQL, shell commands, file paths, or URLs; tools with write access or open `fetch` (SSRF) callable based on model output; agents acting with an admin or service key. Security decisions belong in code, not in the prompt.
- **10 / launch 11 — injection:** model output rendered as HTML or unsanitized markdown (stored XSS through the model).
- **28:** prompts and responses containing user data logged or sent to analytics; providers' data-retention settings vs the privacy policy.
- **12 / 23:** token usage per request logged; cost per user visible; retries on model errors with backoff and a cap (retry storms multiply the bill).
- **26:** what the feature does when the provider is down or rate-limits you — fail soft, not a broken page.

## Security audit — H6 (AI and agents)
_Used by `security-audit.md`. The rule that keeps this honest: **prompt injection alone is not a finding.** It becomes one when attacker-written content reaches another user's model context, triggers an action the attacker couldn't take directly, reveals data they can't read, or feeds a dangerous place (HTML, SQL, shell, a URL). A system prompt telling the model to behave is not a security boundary; only checks in code are._

- **Injection through what the model reads.** Who can write the content the model sees: shared notes, uploaded files, RAG documents, web pages it fetches, emails, issue bodies, tool results? Whose session reads it, and what can that session do? An attacker's shared note that makes the *victim's* assistant call a tool with the victim's powers is the classic. The hole is the missing code check, not the persuasive text.
- **Data bleeding between users.** Conversation history, embeddings, vector search, and prompt caches keyed too broadly. The user or team filter has to be in the query itself, and in every cache key, not just stored on the row.
- **Memory poisoning.** Persistent memory or summaries written from untrusted content, which later steer another user or a more powerful session. Who can create, edit, and delete memory, and is it scoped to one user? Memory a user saved for themselves and only they use is not a finding.
- **Tools with more power than the user.** Tools that run with a service-role or admin key instead of the user's own access; tool arguments from the model going straight into SQL, shell, file paths, or `fetch` (SSRF); destructive tools (delete, send, pay) with no confirmation; agent loops with no step or spend cap.
- **Approvals not tied to the action.** The user approves "summarise my notes" and the agent then sends an email. A confirmation has to name the exact action and arguments, and the code has to execute only what was confirmed.
- **MCP and sub-agents.** Tool descriptions or MCP server metadata treated as policy; two servers offering a tool with the same name; sub-agents inheriting the parent's full credentials; MCP responses trusted as if they were the user.
- **Model output in dangerous places.** Output rendered as HTML or markdown with live links and images (an injected image URL can carry data out in its query string); output used to build SQL, shell commands, file paths, or redirect URLs.
- **Secrets in the context.** API keys, other users' data, or internal notes in the system prompt. "The model won't reveal it" is not a control; assume anything in the context can be extracted.

**Needs a check, usually:** the provider's data-retention setting, the monthly spend cap, and whether production uses the same model and tool config as the repo.

_Adapted from the AI file in Cloudflare's [security-audit skill](https://github.com/cloudflare/security-audit-skill) (MIT, © Cloudflare, Inc.)._
