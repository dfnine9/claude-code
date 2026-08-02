# mixie — RESUME HERE (note to a future session)

Future Claude: this repo carries a **product-concept project called *mixie*** that
was developed in an earlier, now-expired session. There is no chat history to
inherit — everything you need is committed. Read this first, then read
`mixie-concept.md`.

## What this is

*mixie* is a concept for a **strawberry-pink, THC-free / kava-free social tonic
shot** (2 oz clear-glass bottle) that delivers a Brez-style feeling and is safe
for mainstream retail (Target). It is a **product/strategy brief, not code.**

## Where everything lives

| Thing | Location |
|---|---|
| Full concept brief (source of truth) | `mixie-concept.md` |
| Size-comparison artifact source | `mixie-assets/form-factor-scale.html` |
| Product-mockup artifact source | `mixie-assets/mixie-mockup.html` |
| Branch | `claude/tonic-replacement-form-factors-xd55d2` |
| Draft PR | dfnine9/claude-code#2 → https://github.com/dfnine9/claude-code/pull/2 |

**Published artifacts (private to the owner's claude.ai account):**
- Form-factor scale study: https://claude.ai/code/artifact/db987095-1174-42ce-a5ab-84d75c53ec17
- mixie product mockup: https://claude.ai/code/artifact/fdec2086-dd80-4aae-8f6b-831dd3c6a032

## How to pick the work back up

1. Get on the branch (a fresh clone may start on `main`):
   ```
   git fetch origin claude/tonic-replacement-form-factors-xd55d2
   git checkout claude/tonic-replacement-form-factors-xd55d2
   ```
   If PR #2 has already **merged**, that branch is finished — per the session's
   git rules, restart it from the latest default branch and open a *new* PR for
   follow-up work (don't stack onto merged history):
   ```
   git fetch origin main && git checkout -B claude/tonic-replacement-form-factors-xd55d2 origin/main
   ```
2. Read `mixie-concept.md` end to end — it has the locked decisions and the
   open next-steps checklist (§12).
3. Keep committing to `claude/tonic-replacement-form-factors-xd55d2` and pushing
   with `git push -u origin claude/tonic-replacement-form-factors-xd55d2`.

## How to edit / re-publish the visuals in a new session

The HTML sources are in `mixie-assets/`. To **update an existing published
artifact in place (keep its URL)** from a new conversation, call the `Artifact`
tool with BOTH the `file_path` (the edited HTML) AND `url` set to the artifact's
URL above — otherwise a new conversation mints a brand-new URL. Keep the
favicons stable: 🍋 for the scale study, 🍓 for the mockup.

## Decisions already locked

- **Format:** 2 oz clear-glass shot (payload is a non-issue at 60 mL; ~3× headroom)
- **Flavor / color:** Strawberry; pink stabilized with acylated anthocyanins (black carrot / red radish), not strawberry's own fading pigment
- **Name:** *mixie* (working) — NEEDS trademark clearance; "Mixie" collides with a
  South-Asian blender term. Backups: Mixi, Blush, Hum.
- **Signature ingredient:** saffron (affron) mood-lift
- **Focused formula (~1.3 g actives):** lion's mane (clarified) 150 mg · L-theanine
  100 mg · saffron 28 mg · citicoline 250 mg · theobromine 25 mg · magnesium
  glycinate ~350 mg · black seed oil 50 mg (nano-emulsified). Cut: citrulline, ashwagandha.
- **Brand:** wordmark = lowercase heavy sans, i-dots = saffron-gold strawberry seeds.
  Palette: strawberry `#EF456A`, saffron `#F2A63A`, cream `#FFF8F1`, berry ink `#4A0F1E`.

## Likely next asks (from the last session)

- Trademark/domain availability scan on the name
- A COGS / cost-per-shot section (saffron is the swing cost)
- PDF / .docx export of the brief for sharing
- Alternate label directions or a 3-flavor lineup mockup

> All doses/costs in the brief are **conceptual — pending R&D and regulatory review.**
