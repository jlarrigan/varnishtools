# Changelog

## 1.3.0 — 2026-11-02
- **Renamed to VarnishTools** so it's easy to tell apart from the Varnish HTTP cache. The repo is now `jlarrigan/varnishtools` (the old URL redirects) and the plugin is `varnishtools@varnishtools`. **The command is still `/varnish`.** If you installed an earlier version, swap it once: `/plugin uninstall varnish@varnish`, `/plugin marketplace remove varnish`, `/plugin marketplace add jlarrigan/varnishtools`, `/plugin install varnishtools@varnishtools`.
- **New: a dedicated security audit** (`playbooks/security-audit.md`, `/varnish security`). It maps the app first (actors, what's worth taking, every way in, where the decisions are made), then sends focused hunters chosen from the map: obvious things, who can call this, injection, business logic, feature abuse and leaks, AI and agents, and a wildcard, plus teams, spend, supply-chain, and mobile hunters when the stack calls for them. A coverage critic looks for what nobody checked, and a skeptic tries to disprove every finding. Profiles: `quick`, `standard`, `deep`.
- **Three outcomes instead of two.** Findings are confirmed holes; **Needs a check** names the exact fact the repo can't show (a dashboard setting, a host behaviour) and how to check it, with no severity; **Hardening notes** are worth doing but aren't holes.
- **Calibrated security severity** anchors for High / Med / Low, and every fix direction names the rule, the last place to enforce it, and the test that proves it.
- **AI module: a security section** for agents and RAG: injection through what the model reads, data bleeding between users, memory poisoning, tools with more power than the user, approvals not tied to the exact action, MCP trust, and model output in dangerous places.
- **Built for large repos too:** hunters split per subsystem on big apps, several skeptics past ~15 candidates, a retry step for a hunter that comes back cut off, and findings described in plain words rather than as exploit recipes. Tested on a seeded sample app (16 of 16 holes found, 0 false positives) and on a production monorepo.
- **Repo must stay untouched?** Name another place for the report and the run writes nothing in the repo.
- **README: Companion tools** (impeccable and apple-design for design audits, Cloudflare's skill for targets Varnish doesn't cover, `/security-review` on every PR) and a "Which security run?" guide.
- `codebase security` still runs the old bucket's five passes; `everything` is unchanged.
- The method is adapted from Cloudflare's security-audit skill (MIT); see Credits in the README.

## 1.2.0 — 2026-10-02
- **Stack modules** (`playbooks/stacks/`). The core passes are now stack-neutral; Next.js, Vite SPA (incl. Lovable/Bolt/v0), Expo/React Native, Node API, Vercel/serverless, Supabase, Firebase, payments, and AI/LLM modules add each platform's specifics and load only when detected. Passes 17 and 19 now live in the Expo module. A web app run no longer reads mobile material.
- **New: `/varnish status`.** Progress across past runs in `audits/`: came back, decided-but-not-done, open, fixed, ledger triggers, and one recommended next run. Look-only, writes nothing.
- **New: Learn mode.** Add `learn` to any command for a two-line lesson per pattern (the principle, and how to spot it next time) plus a closing "What to take from this run." On by default in `launch`.
- **Works outside Claude Code.** `START.md` lets Cursor, Codex, Copilot, Gemini CLI, and Windsurf run the playbooks, with a snippet for a project's `AGENTS.md` or a Cursor rule.

## 1.1.0 — 2026-10-02
- **New: `/varnish launch`** (`playbooks/launch-review.md`). "Is this safe to put in front of real users?" A plain-English go/no-go for first-time and AI-built apps: twelve checks, a paste-able fix prompt and a how-to-check-it-worked step per finding, a required "couldn't check" list, and a HOLD / FIX THEN SHIP / SHIP verdict.
- **Audits are now strictly look-only.** New Audit-mode rules in the skill override every playbook: no edits outside `audits/`, no git writes, no installs (an audit may be looking at a malicious package), no autofix, no deploys, read-only SQL with plain `EXPLAIN`, and Claude asks before touching any live account. Playbook "Do:" lines are now "Fix direction:", describing the fix for the plan rather than an instruction to act.
- **Real verification.** Findings are checked by a separate subagent that returns confirmed / adjusted / dropped with evidence; every report header shows the tally.
- **Contain today.** Reports and plans put anything exploitable right now (a public secret key, an open table, an open AI endpoint) first, with the containment step in plain words.
- **Plain English.** Every High finding gets a line a non-engineer can act on; plans open with a "Do these today" summary and, for AI-built apps, paste-able fix prompts.
- **Security pass expanded:** routes anyone can call, secrets behind `VITE_` / `EXPO_PUBLIC_` / `REACT_APP_` prefixes, RLS and Firebase-rules specifics, rate limits and AI spend, webhook signatures, injection, and fake or typosquatted packages.
- **Client-direct backends (Supabase, Firebase) are judged as a valid design**, with the RLS policies audited as the authorization layer. An API layer in the middle is still noted as the more scalable default for most growing products.
- Whole-repo scope when there's no usable diff; a red or unrunnable build is recorded as a finding instead of blocking the run.
- Cost check before `everything` and `architecture-review`; `everything` moved to the bottom of the command list.
- Every report and plan ends with a pointer to the author (LinkedIn).
- Fixed: stale tool names (`query_logs`, `@sentry/react-native`, knip in place of ts-prune/unimported, axe tooling), README routing for `architecture-review`.

## 1.0.0 — 2026-07-23
- First public release.
