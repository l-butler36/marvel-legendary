---
name: playtest
description: Use when Paul will manually play the Legendary game to validate an expansion and needs concrete setups to choose from — which mode, scheme, mastermind, villain groups, henchmen and heroes to select on the setup screen, plus what to watch for in each game. Triggers on "/playtest", "playtest setups", "give me playtest setups", "what should I play to test this", "how do I test this expansion by hand", "setups for playtesting", or when an expansion build reaches the user-playtest gate. NOT for Playwright/automated browser runs — that is game-test.
---

# Playtest Setup Generator

Produce a menu of concrete, selectable game setups that a human can play to validate an expansion, each aimed at mechanics the automated gates structurally cannot reach (click-to-defeat board flow, popup clarity, live attack-number math, setup-screen selection).

**Core principle: every name and number in the output is read from a file, never recalled.** The failure mode this skill exists to prevent is confident, plausible, invented setups — hero counts that are wrong, card names that do not exist on the setup screen, and generic "watch for" items not tied to any real fix.

## When to Use

- An expansion build reaches the user-playtest gate (`/new-expansion` Phase 4, pre-merge)
- Paul asks what to play to exercise a new mechanic, or to re-verify after a fix
- A bug is suspected in an area only human play reaches (board clicks, popup wording, feel)

**Do NOT use for:** Playwright-driven verification, repro, or regression — that is `game-test`. This skill produces instructions for a *person*; `game-test` drives a *browser*.

## Step 1 — Ground in data BEFORE composing anything

Read, do not recall. Minimum set:

| Need | Source |
|---|---|
| Card names, effects, attack/VP, keywords | `docs/card-inventory/final/<slug>.md` |
| Exact setup-screen strings | `index.html` in the game root — scope the grep by set (below) |
| What was actually fixed / is fragile | `docs/expansion-specs/<slug>.md`, `docs/audit-results/`, `docs/expansion-progress/<slug>.md` |
| **Cosmetics to exclude — primary** | `docs/expansion-progress/<slug>.md` → **`### Dispositions`** section (pre-triaged per finding: FIXED / CONFIRMED-no-fix / DEFERRED / CLOSED) |
| Cosmetics to exclude — secondary | `docs/known-issues.md` (cross-expansion; most entries are unrelated to this build) |

Setup-screen strings are authoritative for what Paul types/clicks. A hero renamed to avoid a collision (e.g. a set-qualified `"Thor (Asgard)"`) will NOT match the inventory's plain name — always read the literal `value="..."`.

An unscoped `value="` grep returns hundreds of lines across every set. Scope it:

```bash
grep -oE '<input[^>]*data-set="Secret Wars Vol. 1"[^>]*>' index.html
```

Swap the `data-set` value for the expansion in question; drop to `data-set="Core"` for Core picks.

**Reconcile before writing any watch-item.** A finding raised in `docs/audit-results/` may already be dispositioned CONFIRMED-no-fix in the progress doc, or promoted to the base-code branch and fixed weeks later (tracked in `known-issues.md`). Chase the chain audit-results → progress Dispositions → known-issues before turning a finding into a watch-item. Flagging an already-disproven bug wastes Paul's play session.

## Step 2 — Derive the mode math, do not assume it

This is a **two-file cross-read**: the function in `script.js` is meaningless without the value the scheme actually stores in `cardDatabase.js`. Read both.

- **Hero count** — `getEffectiveHeroCount()` in `script.js`, plus the scheme's `requiredHeroes` in `cardDatabase.js`. **Gotcha:** a scheme storing the What If? default is indistinguishable from "unset", so Golden Solo silently upgrades it to the Golden default. Reading `requiredHeroes: 3` and reporting "3 heroes in Golden Solo" is the classic wrong answer.
- **Villain groups + auto-locks** — `getEffectiveSetupRequirements()` in `script.js`, plus the scheme's `requiredVillains` / `extraVillainGroups`. Golden Solo derives its own count and locks the mastermind's `alwaysLeads`; What If? Solo uses the scheme value and ignores `alwaysLeads` entirely.
- **Which slot locks** — `alwaysLeadsType` on the mastermind is either `"villain"` or `"henchmen"`. It decides whether the auto-lock consumes a *villain* slot or a *henchmen* slot. A henchmen-type mastermind (e.g. Dr. Doom) leaves both villain slots free — useful when you want an isolated test with no expansion villains in the way.

Getting these wrong makes every setup unusable — the setup screen refuses to start the game.

## Step 3 — Build the watch-list from real findings

Each "watch for" line must trace to something concrete: a fixed bug, a frozen spec assertion, an audit finding, a rulebook ruling, or an improvised UI whose clarity is genuinely unverified. Never pad with generic QA filler ("check the game runs correctly").

Mark the subjective ones as subjective — improvised UI needs Paul's gut reaction, not a pass/fail.

## Step 4 — Compose setups that cluster mechanics

- **Order by importance.** Setup 1 exercises whatever was actually game-breaking. Say plainly that it is the one to play if only one game happens.
- **Isolate first, combine later.** For the highest-priority mechanic, pick a quiet scheme (often a Core one) so a failure is unambiguous. Save expansion schemes that crowd the board for later setups.
- **Choose selections that force the mechanic to fire.** Pick the mastermind/villain groups that maximise the target's frequency; require the specific hero a fix depends on.
- **Cover both modes.** At least one Golden Solo and one What If? Solo setup — mode divergence is invisible to the Golden-only validator.
- **State why each combination was chosen** in one italic line, so Paul can substitute knowingly.

## Output Format

Inline in chat (not a file or artifact) unless Paul asks otherwise.

1. **One-line orientation** — where to play (URL/branch) and how many setups follow.
2. **Broad strokes** — which setup matters most and why, before any detail.
3. **🚩 STOP AND TELL ME** — global red flags meaning a fix did not hold. Put this ABOVE the setups; it is the highest-value content and must not be buried.
4. **Per setup:**
   - `## Setup N — Short Name` (⭐ mark the priority one)
   - A two-column table: **Mode / Scheme / Mastermind / Villains / Henchmen / Heroes (N)** — exact setup-screen strings, note which groups auto-lock
   - One italic line: *why this combination*
   - **Watch for:** bulleted, bolded key terms, plain-language mechanics
5. **🙈 Already known — don't report** — logged cosmetics, so Paul does not waste attention.
6. **Close on one blunt line** — what a pass on Setup 1 actually means.

Plain language throughout: "the villain closest to the villain deck grabs it", not "rightmost city-index capture". Paul is non-technical.

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Recalling card/hero names instead of grepping `index.html` | Names that do not exist on the setup screen; setup impossible to follow |
| Assuming standard hero/villain counts | Setup screen blocks game start; wasted session |
| Reading `requiredHeroes` without applying the mode default | Wrong hero count in Golden Solo (the default-vs-unset trap) |
| Forgetting `alwaysLeads` auto-locks a slot in Golden Solo | Paul is told to pick a group the UI has already locked |
| Ignoring `alwaysLeadsType` | Wrong slot assumed locked; a clean isolation setup is missed |
| Generic watch-items not tied to a real fix | Paul cannot tell a genuine failure from normal play |
| Watch-item for a finding already dispositioned no-fix | Paul hunts a bug that was disproven weeks ago |
| Omitting the known-cosmetics list | Paul reports already-logged noise |
| Burying red flags below the setups | The one thing that must be seen gets skimmed past |
| Producing a file/artifact when asked for a list | Paul reads inline; extra surfaces get ignored |

## Red Flags — stop and re-ground

- About to write a card, hero, scheme or mastermind name you did not read from a file this session
- About to state a hero count from the rules summary rather than the function **and** the scheme's stored value
- A "watch for" bullet you cannot trace to a fix, spec, audit finding, or rulebook line
- A "watch for" bullet you have not checked against the progress doc's `### Dispositions` section

All of these mean: go back to Step 1.
