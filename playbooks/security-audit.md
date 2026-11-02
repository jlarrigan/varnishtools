# Security Audit

_"Can someone do something in this app they shouldn't be able to?" The dedicated, deep security pass: map who can reach what, send focused hunters after each kind of hole, prove every finding against the code, and say plainly what wasn't covered. Part of VarnishTools._

## ▶️ How to run this
**In the target repo's Claude session:** `/varnish security` (or _"Read this file and run a security audit on this repo."_). Add a profile if you want one: `/varnish security quick` · `/varnish security deep`. Add `learn` for a lesson per finding. Add paths to scope it: `/varnish security app/api/billing`.

**Repo must stay untouched?** If the user says so (or the repo isn't theirs to write to), write the report wherever they name instead of `audits/`, and say so in the header.

Claude will: detect the stack and load its modules → **map** the app (Step 1) → pick **hunters** from the map and run them **look-only** (Step 2) → a **coverage critic** looks for what nobody checked (Step 3) → a separate **skeptic** tries to disprove every finding (Step 4) → write `audits/<YYYY-MM-DD>-security-<HHMM>.md` → show you the headline.

The Audit-mode rules in `skills/varnish/SKILL.md` apply in full (no edits outside `audits/`, no git writes, no installs, read-only SQL, ask before touching any live account). This playbook adds one more: **never send traffic to a deployed site, a real account, or a paid API to prove a finding.** Evidence comes from source, config, git history, migrations, and build output already on disk.

**When to run it.**
- After `/varnish launch` says ship and real users are arriving.
- Before adding teams/workspaces, payments, credits, file sharing, or an AI agent with tools.
- After a big change to sign-in, roles, or the data model.
- On an inherited codebase, or before a security questionnaire.
- First time shipping? Run `/varnish launch` first. It answers the question you actually have, faster.

**Cost.** Say this in one line before starting, and confirm on `deep`:

| Profile | What runs | Good for |
|---|---|---|
| `quick` | map → H1 + H2 → skeptic. No critic round; gaps are listed as not covered. | small apps, re-runs, a fast first look |
| `standard` (default) | map → every hunter the map calls for → one critic round → skeptic. About 10 agents on a small app; a large monorepo needs 15-20 (split hunters per subsystem). | most apps |
| `deep` | hunters split per subsystem → critic rounds until one comes back clean → skeptic | larger apps, money or health data, before a big launch |

Profiles change how much ground is covered, never the evidence bar.

## The rules every hunter follows
Copy this block into every hunter's prompt, word for word.

```
1. A finding needs a victim. Name the attacker (stranger, signed-in user, member of
   another team, low-role member), what they get (whose data, which action, whose
   money), and the path through the code as a file:line chain. No victim, no finding.
2. Hardening notes are not vulnerabilities. If one layer already blocks the attack,
   a missing second layer is a hardening note. Exception: a secret key that is or
   was public (client code, a public env prefix, git history) is always a finding.
3. Don't guess what lives outside the repo. When the deciding fact is a dashboard
   setting, a host or proxy behaviour, a provider config, or production data, record
   it as NEEDS A CHECK with the exact missing fact and how the owner can check it.
   No severity on these.
4. Users acting on their own data or account is not a finding, and neither is an
   admin using admin powers. Getting something you'd otherwise pay for, or getting
   past a limit the app sets on you (credits, plan, quota, a pinned column), IS a
   finding, even though it happens on your own account.
5. Claim only what the code shows. If you can't say the concrete damage, it is
   smaller than it feels.
6. Read, don't run. Don't execute the app's code, send requests to deployed URLs,
   or call paid APIs. Read the code, config, migrations, git history, and any
   build output already on disk. Looking up public advisories (npm, GitHub
   advisories) is fine; it isn't traffic to the app. With no lockfile, versions
   come from package.json ranges; say so.
7. Verify every "this is safe because" comment and every check you rely on. Follow
   the call to the function that actually enforces it.
8. Return results in the shape below, including what you checked and found clean.
9. Rate the chain the repo gives the attacker, not the finding in isolation. If a
   finding needs a prerequisite (an id, a user id, a workspace id) and another
   finding hands it to the same attacker, rate it as the chain and name the link
   ("ids are unguessable, but /api/search lists them: see F05").
10. Committed or hardcoded keys count as real unless they are provably dummies
   (a literal FAKE, all zeros). Either way, add "is this key live?" to Needs a check.
   Don't open gitignored secret files (.env, keychains) to prove a key is live, and
   never copy a secret value into your results: cite file:line and the key type.
12. Describe, don't weaponise. Say which code path lets which kind of user reach
   what, in plain words. No payloads, no request recipes, no step-by-step exploit.
   Keep each entry short and write results as you go, so a cut-off loses little.
11. A known-vulnerable dependency is a finding only if the app uses the vulnerable
   feature in a way the code shows. Otherwise it goes to Needs a check.
```

**Hunter return shape** (one entry per finding):
`id` · `class` · `claim` (one sentence) · `who · could reach what · through which code` (a sentence, not a recipe) · `path` (file:line chain from way-in to the harm) · `evidence` (quoted lines) · `proposed severity` · `cost of doing nothing` (High when it's reachable now and gets worse as users and data grow; Low when it stays inert) · `chain` (other findings that supply its prerequisites, if any) · `needs-a-check fact` (if the decision hinges on something outside the repo) · then a **Checked and clean** list: every way-in or check this hunter covered with no finding.

## Step 1 — Map the app (before any hunting)
Ten minutes of mapping is what turns a checklist into an audit. Answer five questions and write the answers into the report's **Map** section:

1. **What runs where?** Browser, server routes, server actions, edge/cloud functions, the database (and whether the browser talks to it directly), background jobs, third parties. Load the stack modules (`stacks/_index.md`).
2. **Who are the actors?** Anonymous visitor, signed-in user, paying user, team owner / admin / member, staff, webhooks and other services, scheduled jobs, and the AI model itself if there is one. Each role the app distinguishes is its own actor.
3. **What's worth taking?** Other users' data, payment state, credits and plan entitlements, admin powers, anything that spends money (AI, SMS, email), secrets, and the ability to delete things.
4. **Every way in.** Routes, server actions, functions, tables the client can reach directly, storage buckets, webhooks, uploads, imports, share links, invites, search, exports, cron, queue consumers, deep links, and AI inputs (prompts, retrieved documents, tool results).
5. **Where are the decisions made?** Every place the app decides *who you are* and *what you're allowed to do*: middleware, a route-level check, an RLS policy, a database function, a wrapper. Two places that make the same decision differently is where holes live.

The output is one table, which is also the coverage list in the report:

| Way in | Who can reach it | What it touches | Where it's checked |
|---|---|---|---|
| `GET /api/notes/[id]` | anyone | one note, by id | nowhere ← |

**Prior runs.** Read earlier `audits/*-security-*.md` reports (their Coverage sections) and `audits/*-plan-*.md` (the Accepted-Risk Ledger). Aim this run at what wasn't covered last time and what changed since (`git log` on those paths). Accepted items aren't re-litigated, per `_report-shape.md`.

## Step 2 — Hunt
Each hunter is a **fresh subagent**. Give it the Map, its brief below (pasted), the rules block (pasted, word for word), and the paths of the loaded stack modules and of `codebase-audit.md` so it reads them itself. IDs: H1-H7 below; the add-ons are HT (teams), HS (spend), HC (supply chain), HM (mobile). No subagents in your tool? Run each hunter as its own deliberate pass, re-reading the rules block before each one, and say so in the header. Run them in parallel. **H1 and H2 always run**; the rest run when the map gives them something to hunt. Mark the others N/A with a reason. Split H2-H5 per subsystem (API, database, web, mobile) whenever one hunter's scope wouldn't fit comfortably in one context: always on `deep`, and on `standard` for monorepos and large apps. Record the split in the header. **A hunter came back cut off or empty?** Re-run it once, narrower and terse (rule 12), and note it in the header; if it fails again, list its scope under Not covered.

**H1 · Obvious things (always).** Literal, not creative: check every item and report each result. Run the security-tagged passes in `codebase-audit.md` (3, 10, 15, 20, 28) and then:
- grep for hardcoded secrets: `password`, `secret`, `apikey`, `token`, `Bearer`, `-----BEGIN`, provider key formats (stack modules list them)
- security TODOs: `TODO: auth`, `FIXME: validate`, `HACK`, commented-out checks
- debug or dev modes that production can switch on through an env var, query param, or header
- seed or test accounts and passwords that would work in production
- unprotected `/debug`, `/admin`, `/test`, `/metrics`, `/env`, `/config` routes
- committed `.env`, `*.pem`, `*.key`, `credentials.json`, and whether `.gitignore` covers them (`git log --all -- <file>` for ones since deleted)
- `eval`, `new Function`, `child_process`, `exec`, dynamic `import()` with input in them
- CORS set to `*`, or a reflected origin combined with credentials
- session cookies without `HttpOnly`, `Secure`, `SameSite`
- open redirects: `redirect`, `next`, `returnTo`, `url`, `callback` params used without an allow-list
- production errors that return stack traces, SQL errors, or internal paths

A flag isn't a finding until you trace the impact: a cookie without `HttpOnly` matters only if it holds something sensitive and JavaScript doesn't need it.

**H2 · Who can call this (always).** For every way in on the map:
- Is there any check of who's calling? A check that only hides a button, or lives only in middleware the route can be reached around, doesn't count.
- Does it check the caller owns or belongs to *this* resource (`/api/notes/123` → is note 123 theirs, is it in their workspace)?
- Is there a second path to the same change with a weaker check (an API route and a server action, a REST route and a direct table write)?
- Can a field in the request override what the check intended: `user_id`, `role`, `workspace_id`, `price`, `plan`, `is_admin` taken from the body or query?
- Signed-in but not authorised: routes that check "logged in" when the data is per-user or per-team.
- Bulk, batch, export, and import routes: is the check per item, or once for the batch?
- Client-direct backends: RLS or security rules per table **and per operation** (select / insert / update / delete), storage bucket policies, `SECURITY DEFINER` functions, views (see the stack module).
- Webhooks: signature verified before anything is trusted (pass 15).

**H3 · Injection (when input reaches a dangerous place).** Trace untrusted input to: SQL or NoSQL built from strings; HTML (`dangerouslySetInnerHTML`, `v-html`, unsanitised markdown); shell commands; file paths (`../`, uploads named by the user); redirects; template engines; server-side fetches of a user-supplied URL (SSRF, including webhook and avatar URLs). Then the indirect paths: data stored safely and later used raw somewhere else (a display name rendered as HTML in an admin view, a slug that becomes a file path), and injection through field names, keys, and headers, not just values.

**H4 · Business logic (when there's money, credits, limits, or multi-step flows).** Scanners can't find these, and they're the expensive ones:
- Skip a step, go back a step, or replay a finished step (re-use a one-time code, re-submit a completed checkout).
- Race two requests: check-then-act without an atomic update gives double redemptions, double spends, over-limit usage.
- Numbers: negative, zero, huge, decimal where an integer was assumed, a string where a number was expected.
- Partial failure: step 2 of 3 fails. Is the user left with the paid thing and no charge, or charged with nothing?
- Time: trial and expiry boundaries, time zones, a client-supplied timestamp.
- Default posture: what happens when a config value is missing, a feature flag is off, or the payment or AI provider is down? Does it fail open?
- Paid features checked only in the browser, or a plan column the user can write (overlaps launch check 4).

**H5 · Feature abuse and leaks (when there's sharing, search, export, invites, or notifications).** Legitimate features used for something else:
- Export or backup that includes other users', deleted, or draft data.
- Import or restore that skips the normal validation or writes where the user can't.
- Search, filter, or sort that reveals records the user can't open ("3 results" for a private title).
- Different answers for "no such account" and "wrong password" on sign-in, reset, signup, or invite: user enumeration.
- Share, preview, or invite tokens that are guessable (sequential, a timestamp, base64 of the id), never expire, or unlock more than one item.
- Private responses a CDN could cache and serve to someone else.
- A notification or webhook URL the server will fetch (SSRF).
- Logs, analytics, error tracking, and AI providers as a second way to read user data (pass 28).

**H6 · AI and agents (when the `ai-llm` module loads).** Run the AI module's security-audit section: injection through content the model reads, data bleeding between users, memory poisoning, tools with more power than the user, approvals not tied to the exact action, MCP trust, and model output reaching HTML, SQL, shell, or URLs.

**H7 · Wildcard (standard and deep).** No category; find what the others won't:
- The strangest code in the repo. Why does it exist, and what happens if it's abused?
- Half-finished, experimental, or "legacy" features. They had the least review.
- API calls the UI never makes. The UI limits users; the API doesn't.
- Routes, params, or headers nobody documented.
- Features that weren't designed to meet: sharing + caching, import + webhooks, impersonation + API keys.
- Git history: reverted security fixes, removed checks, secrets committed and deleted.
- What the tests *don't* test, compared with what they do.

**Add by stack** (run as their own hunter when the map calls for it):
- **HT · Teams and workspaces** (more than one user sharing data): the team filter in every query, cache key, search index, and signed URL; ids that collide across teams; soft-deleted rows still readable; restore putting back access someone lost; old sessions still working after removal from a team.
- **HS · Spend and overload** (paid APIs, uploads, heavy work): expensive work done before sign-in; quotas that reset early or count the wrong thing; unbounded uploads, page sizes, and regexes; retry loops that multiply provider bills.
- **HC · Supply chain and CI** (public repo, GitHub Actions, published packages): pass 3 in depth, plus workflow secrets exposed to pull requests from forks (`pull_request_target`), unpinned third-party actions, and `postinstall` scripts.
- **HM · Mobile** (Expo/React Native): deep links that act without confirming, secrets in the app bundle, webviews with a JavaScript bridge (see the Expo module).

## Step 3 — Coverage critic (standard and deep)
One fresh agent gets the Map, every hunter's **Checked and clean** list, and the findings list (not the hunters' reasoning). Its only job: **what did nobody check?** A way in no hunter touched, an actor/resource pair nobody tested, a hunter the map called for that didn't run. Each gap becomes a short brief for one more hunter round. `standard`: one critic; if it names gaps, one extra hunter round on them, and whatever is still open goes under Not covered. `deep`: critic and extra round repeat until a critic comes back clean. `quick`: no critic; every gap the map shows goes under Not covered.

## Step 4 — Verify (a separate skeptic)
More than about 15 candidates? Split them across two or three skeptics by area; each is still fresh and sees only its own candidates. Findings from the critic's extra round go through a skeptic too, and count in N.

**First, dedupe.** Hunters overlap. Merge entries that share a root cause and location into one candidate (keep every hunter's evidence), number them `C01, C02, …`, and record both counts. **"N raw" in the tally is this deduplicated candidate count.** The header also says how many hunter entries went in ("62 hunter entries → 26 candidates").

Hand every candidate (ID, claim, path, chain) to a **fresh subagent** whose job is to disprove it. It checks:
- Is the check enforced somewhere upstream (middleware, RLS, a database function, a wrapper, a later migration)?
- Is the key public by design (Supabase anon / publishable, Stripe `pk_`, Firebase web config)?
- Is the "attacker" already allowed to do this?
- Does the exposed data actually contain anything sensitive?
- Does the decision hinge on something the repo can't show? If so, it moves to Needs a check.
- Is the severity right for the chain the repo gives the attacker (rule 9), not for the finding alone? Marking a link down because "the id is unguessable" is wrong when another confirmed finding hands out the id.

It returns `confirmed / adjusted / needs-a-check / dropped` with the quoted evidence, and may add a finding it trips over. Before keeping a verifier-added finding, re-read its cited lines yourself; it had no skeptic of its own. Tally in the header, on the report's second line: **"N raw → M kept (K dropped, J adjusted, C moved to needs-a-check, A added by the verifier)."** Merges count as adjusted.

## Severity, calibrated
Varnish keeps three levels (`_report-shape.md`). For security, anchor them like this:

- **High**: a stranger or an ordinary user fully defeats a real protection, with real consequences. They read or change other users' data, take over accounts, get paid features or money for free, run code on your server, or spend your money with no limit. A secret key that is or was public.
- **Med**: a real break with limited reach. Narrow data, unusual preconditions, needs an insider or privileged account, or a limit that exists but is weak.
- **Low**: leaks non-secret internals, reveals which emails have accounts, or takes a lot of effort for little gain.

The deciding question between High and Med: does it **fully defeat** a protection on something that matters, or only **weaken** it? If you can't say the concrete damage, it's lower than it feels.

## The report
`audits/<YYYY-MM-DD>-security-<HHMM>.md`, in the Audit shape (`_report-shape.md`), with these additions:

1. **Header** also carries the profile, stack modules, and the hunters that ran.
2. **🚨 Contain today**, as in the shared shape. Keys that look like placeholders still go here, with "if this key is real" in front.
3. **Findings headline.**
4. **The Map**: the Step 1 table, trimmed to what matters.
5. **Findings, grouped by area.** Each finding also gets a **Who · gets · how** line: _"A signed-in user · any other user's notes · changes the id in `GET /api/notes/[id]`, which never checks the owner (`app/api/notes/[id]/route.ts:8`)."_ The direction line names **the rule the code must enforce, the last place it should be enforced** (the server route, the RLS policy, the database function, never only the client), **and the test that proves it** ("a second account gets a 404").
6. **Needs a check.** Each item: the exact fact the repo can't show, and how to check it (a dashboard setting, a question for the host, a two-account test on a staging copy). No severity. Never empty if anything depended on a dashboard. Stack modules call these "Couldn't check"; in this report they all go here.
7. **Hardening notes.** One line each, no tags. Worth doing; not holes.
8. **Coverage.** Every way in from the Map with the hunters that covered it, then **H1 checklist results** (one line per item), then **Not covered this run** (what the repo couldn't show, and any gap the critic named that no hunter closed). Close it with: _"One run typically finds about half of what repeated runs find. After fixing, run `/varnish security` again; it aims at what this run didn't cover."_
9. **→ Handoff**, version stamp, author footer.

## What not to report
1. A checklist miss with no victim.
2. A missing second layer when the first already blocks the attack (→ hardening note).
3. A guess about the host, provider, proxy, or dashboard (→ needs a check).
4. Users doing things to their own data, or admins using admin powers.
5. More impact than the code shows.
6. Prompt injection with no capability behind it (see the AI module).
7. Publishable keys flagged as secrets.
8. A severity on a needs-a-check item.
9. Items already in an Accepted-Risk Ledger, unless something changed.

---
_v1, 2026-11-02. The method (map first, hunters chosen from the map, an "obvious things" sweep, a coverage critic, three outcomes, calibrated severity, fix at the last decision point) is adapted from Cloudflare's [security-audit skill](https://github.com/cloudflare/security-audit-skill) (MIT, © Cloudflare, Inc.), rewritten for Varnish's look-only, plain-English, findings-first shape. Part of VarnishTools._
