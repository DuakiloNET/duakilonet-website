---
title: "Git Rebase vs Merge: Stop Guessing, Start Choosing"
date: 2026-06-13
description: "Both merge and rebase integrate changes between branches — but they leave very different histories. Here's exactly when to use each, and why it matters."
tags: ["git", "workflow", "version-control", "tutorial"]
categories: ["tools"]
draft: false
showHero: false
showTableOfContents: true
showReadingTime: true
---

If you've used Git for more than a week, you've run into this moment: you're on a feature branch, the main branch has moved ahead, and you need to bring in the latest changes.

Two commands can do it. `git merge`. `git rebase`. They both work. But they're not the same — and choosing wrong can cause real headaches for your team.

Let's break it down.

## What They Actually Do

### git merge

Merge takes two branch tips and creates a new "merge commit" that ties them together. Your history reflects exactly what happened: two lines of work converged at a specific point.

```bash
git checkout feature
git merge main
# Creates a new commit: "Merge branch 'main' into feature"
```

The result looks like this:

```
A - B - C - D  (main)
     \       \
      E - F - G  (feature, after merge)
```

History is preserved in full. Nothing is rewritten.

### git rebase

Rebase takes your commits and *replays* them on top of another branch. The commits get new hashes — they're technically new commits, but with the same changes.

```bash
git checkout feature
git rebase main
# Replays E and F on top of D
```

The result:

```
A - B - C - D  (main)
              \
               E' - F'  (feature, after rebase)
```

Clean, linear, as if you'd started your feature branch from D all along.

## The Real Tradeoffs

| | Merge | Rebase |
|---|---|---|
| History | Complete, shows branching | Linear, looks cleaner |
| Commit hashes | Preserved | Rewritten |
| Safe on shared branches | ✅ Yes | ❌ No |
| Conflict resolution | Once, at merge | Per commit replayed |
| Blame/bisect clarity | Can get noisy | Cleaner |

## When to Use Each

### Use merge when:

**The branch is shared.** If anyone else has based work on your branch, rebasing rewrites history they depend on. This is the golden rule — don't rebase public branches.

**You want to preserve context.** Merge commits document *when* integrations happened. On long-running projects, this audit trail has real value.

**You're integrating a completed feature into main.** A merge commit here is appropriate — it marks the moment the feature landed.

### Use rebase when:

**You're cleaning up local work before pushing.** If your commits are a series of "wip", "fix", "oops", "actually fix" — rebase interactive (`git rebase -i`) is your best friend. Squash, reorder, clean up before anyone else sees it.

**You want to bring your feature branch up to date with main.** Rebasing your local feature branch onto main keeps history linear and avoids unnecessary merge commits cluttering the log.

```bash
# Bring feature branch up to date with main
git fetch origin
git rebase origin/main
```

**You're the only one on the branch.** No risk of disrupting anyone else.

## The Rule That Matters

> **Never rebase commits that exist outside your local repository and that others may have based work on.**

If you force-push rebased commits to a shared branch, collaborators will have history that diverges from yours. Their future pulls will surface confusing duplicates. You'll create work for everyone.

The practical guideline: **rebase locally, merge publicly.**

- ✅ Rebase your local feature branch onto main before pushing
- ✅ Merge feature branches into main with a PR/merge commit
- ❌ Never `git push --force` rebased commits to `main`, `dev`, or any branch others use

## A Workflow That Works

Here's the pattern I use on most projects:

```bash
# Starting new work
git checkout -b feature/my-thing

# ... make commits ...

# Before opening PR: clean up and update
git fetch origin
git rebase origin/main       # bring up to date
git rebase -i origin/main    # optional: clean up commit messages

# Push
git push origin feature/my-thing

# After PR approval: merge (not rebase) into main
```

This gives you the best of both: clean local history, and preserved merge record on the main branch.

## One More Thing: Interactive Rebase

If you only take one thing from this post, let it be `git rebase -i`.

```bash
git rebase -i HEAD~3   # review last 3 commits
```

This opens an editor where you can:
- `squash` — combine commits
- `reword` — change commit messages
- `drop` — delete commits entirely
- `reorder` — rearrange commit order

It turns messy work-in-progress history into a clean narrative before it ever reaches anyone else.

---

Neither merge nor rebase is universally better. The difference is *context*. Once you internalize that, the choice becomes obvious every time.
