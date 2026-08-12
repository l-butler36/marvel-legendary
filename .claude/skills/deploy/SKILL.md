---
name: deploy
description: Push current branch to master and deploy to GitHub Pages. Runs a pre-push checklist, pushes, and reminds user to verify the deployment.
disable-model-invocation: true
---

# Deploy to GitHub Pages

Follow these steps in order every time.

## Step 1 — Pre-push checklist

Run `git status` and confirm:
- No unintended files are staged
- No sensitive files (`.env`, credentials) are included
- You are on the `master` branch (run `git branch --show-current` to confirm)

Also check for leftover debug logs in the shipped game files (baseline is zero — player messages use `onscreenConsole.log`, not `console.log`):
```
grep -rnE "console\.log\(" Legendary-Solo-Play-main/Legendary-Solo-Play-main/script.js Legendary-Solo-Play-main/Legendary-Solo-Play-main/cardAbilities*.js Legendary-Solo-Play-main/Legendary-Solo-Play-main/expansion*.js | grep -vE ":[0-9]+:\s*(//|\*)"
```
Expected: no output. Any hits are debug `console.log(` traces that shouldn't ship — flag them to the user to remove before pushing.

If anything looks unexpected, stop and flag it to the user before continuing.

## Step 2 — Service Worker cache sync (CRITICAL — now automated)

Forgetting this is the #1 cause of "changes don't show on GitHub Pages." A tool now handles it — no more hand-bumping `CACHE_NAME` or hand-adding files to `FILES_TO_CACHE`. Run:
```
"C:\Program Files\nodejs\node.exe" tools/sync-sw-cache.js
```
(Bare `node` isn't on the Bash PATH on this machine — use the full path above.)

This does two things in `sw.js`: bumps `CACHE_NAME` (`legendary-vN` → `v{N+1}`) so browsers/iPads re-cache, and rebuilds `FILES_TO_CACHE` to exactly mirror every file shipped under the game root (so a new expansion JS or card-art file is never left un-cached). It prints what it added/removed.

Then:
- If `sw.js` changed, commit it: `git add Legendary-Solo-Play-main/Legendary-Solo-Play-main/sw.js && git commit -m "chore: bump SW cache"`. This commit joins the push.
- Read the tool's ADDED/REMOVED report out loud. Additions of real shipped files are expected. If it reports a **REMOVED** path, that's a file deleted from disk but still referenced — confirm that deletion was intended before continuing.

To preview without writing, run with `--check` (reports drift, exits 1 if stale, does not bump or write).

## Step 3 — Show what's being pushed

Run `git log origin/master..HEAD --oneline` to show the user which commits will be pushed (now including the cache-bump commit from Step 2). Read the list out loud in plain English so the user can confirm they're happy with it.

If there are no commits ahead of origin/master, tell the user there is nothing to push and stop.

## Step 4 — Push

Run:
```
git push origin master
```

Report whether it succeeded or failed. If it failed, show the error and stop — do not retry without diagnosing the cause.

## Step 5 — Remind user to verify

Tell the user:
- GitHub Pages takes ~1 minute to deploy after a push
- To verify: open **https://l-butler36.github.io/marvel-legendary/** in their browser
- Use **Ctrl+Shift+R** (hard refresh) to bypass the browser cache and see the latest version
- If the page looks unchanged after 2 minutes, check the Actions tab on GitHub for deployment errors
