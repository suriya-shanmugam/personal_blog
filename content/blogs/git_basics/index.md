+++
title = 'Git: Building the Right Mental Model'
date = 2026-03-28T15:53:27-07:00
draft = false
+++
If you’ve ever forcefully deleted a repository and re-cloned it just to fix a merge conflict, you are not alone. Git is notorious for its steep learning curve. The problem isn't that Git's commands are too complex; it's that most developers learn the commands without understanding the underlying **data model**. 

Once you stop treating Git like a black box of magic commands and start seeing it as a beautifully structured graph, everything clicks. Let's break it down.

---

### What is Git?

At its core, Git is a **Distributed Version Control System (DVCS)**. It allows multiple people to work on the same codebase simultaneously, tracks every modification, and lets you travel back in time to any previous state of your project. 

Unlike older systems that stored a base file and a list of changes (deltas), Git thinks of your data more like a series of **snapshots**. Every time you commit, Git essentially takes a picture of what all your files look like at that exact moment and stores a reference to that snapshot.

---

### The Git Architecture & Mental Model

To truly master Git, you need to understand two key architectural concepts: the **Three Trees** and the **Graph**.

#### 1. The Three Trees (Local Architecture)
When you work on a repository locally, your files move through three distinct phases:

1.  **The Working Directory:** This is your current workbench. It contains the actual files you are editing right now.
2.  **The Staging Area (The Index):** Think of this as a loading dock. You purposefully move modified files here (`git add`) to prepare them for your next commit.
3.  **The Local Repository:** Once you commit (`git commit`), Git takes everything on the loading dock and stores it permanently as a snapshot in your history.

#### 2. The Mental Model: The Directed Acyclic Graph (DAG)
Git history is not a simple list; it's a tree-like graph. 
* **Commits** are nodes (circles) in the graph. Each commit points to its parent(s).
* **Branches** are not physical containers of commits. They are simply **lightweight, movable pointers** stuck to specific commits.
* **`HEAD`** is a special pointer that tells Git exactly where you are sitting in the graph *right now*. Usually, `HEAD` points to a branch name.

**Base State Graph:**
You are working on `feature`, which is three commits ahead of `main`.

```text
 (A) --- (B) --- (C)  <-- [main]
          \
          (D) --- (E) --- (F)  <-- [feature] <-- [HEAD]
```

---

### Basic Commands: The Merge Graph

Let’s look at standard branching and merging. When you are on `main` and run `git merge feature`, Git creates a **new merge commit** (M) that has two parents, tying the histories together.

**Initial State:**
```text
 (A) --- (B) --- (C)  <-- [main] <-- [HEAD]
          \
          (D) --- (E)  <-- [feature]
```

**Action:** `git merge feature`

**Resulting Graph:**
A new commit (M) is created, combining changes. The `[main]` pointer moves to M.

```text
 (A) --- (B) --- (C) --- (M)  <-- [main] <-- [HEAD]
          \             /
          (D) ------- (E)  <-- [feature]
```

---

### Undoing Mistakes: Git Reset vs. Git Revert

Both commands undo changes, but they affect the graph in entirely different ways.

#### `git reset` (Rewriting History)
Reset is used for **local, unpushed** changes. It picks up the current branch pointer and moves it backward in time.

**Initial State:** You realize commit (C) is broken and you want to erase it.
```text
 (A) --- (B) --- (C)  <-- [main] <-- [HEAD]
```

**Action:** `git reset --hard B` (Moves pointer to B and clears working directory).

**Resulting Graph:**
The `[main]` pointer moves back to B. Commit (C) is left behind. It is now orphaned and will eventually be deleted by Git's garbage collection. History has been rewritten.

```text
 (A) --- (B)  <-- [main] <-- [HEAD]     [ (C) orphaned ]
```

#### `git revert` (Adding to History)
Revert is used for **shared remote** branches. It does not erase commits; instead, it figures out the exact opposite of a specific commit and adds a **new commit** performing that undo action.

**Initial State:** Commit (C) is broken, but you already pushed it to the remote. You can't use `reset`.
```text
 (A) --- (B) --- (C)  <-- [main] <-- [HEAD]
```

**Action:** `git revert C`

**Resulting Graph:**
Git creates a new commit (`C_Opposite`) which cancels out the changes made in (C). The history flows forward cleanly.

```text
 (A) --- (B) --- (C) --- (C_Opposite)  <-- [main] <-- [HEAD]
```

---

### Detached HEAD and Relative Refs

#### The Detached HEAD Graph
Normally, `HEAD` points to a branch name (e.g., `[main]`). If you check out a specific commit hash (e.g., `git checkout B`), `HEAD` detaches from the branch and points directly to the commit.

**Initial State:**
```text
 (A) --- (B) --- (C)  <-- [main] <-- [HEAD]
```

**Action:** `git checkout B` (detached `HEAD` state). If you make a new commit (X) here, it branches off into a void.

**Resulting Graph:**
```text
           [HEAD]
             |
            (X)  <-- (New orphaned commit)
           /
 (A) --- (B) --- (C)  <-- [main]
```
*(If you switch back to `main` now, commit X is lost. To save it, you must create a branch right there: `git switch -c recovery`).*

#### Relative References Graph (`~` and `^`)
Popularized by *Learn Git Branching*, relative references make navigating the graph easy. Assume this is our graph with `HEAD` at commit (D):

```text
 (A) --- (B) --- (C)  <-- [main]
          \
          (D) <-- [feature] <-- [HEAD]
```
* `HEAD~1` (Parent) = **(B)**
* `HEAD~2` (Grandparent) = **(A)**

If (D) were a merge commit with parents (B) and (C):
* `HEAD^1` (First parent, straight back) = **(B)**
* `HEAD^2` (Second parent, the branch merged in) = **(C)**

---

### The Ultimate Safety Net: Git Log and Reflog

* **`git log`:** Shows the official history of the current branch, tracing parent nodes backward.
* **`git reflog` (Reference Log):** The "time travel" insurance policy. `git log` only shows active commits. `git reflog` lists every single movement of the `HEAD` pointer on your local machine, whether you switched branches, reset, or committed. Even if you `--hard` reset a branch and seemingly lose all your work, you can find the lost commit hash in the `reflog` and reset right back to it.

---

### Advanced Superpowers: Cherry-Pick and Interactive Rebase

#### Git Cherry-Pick Graph
Cherry-pick acts like a cut-and-paste tool. It allows you to grab a specific commit from one branch and apply it to another.

**Initial State:** You are on `main` and want the specific hotfix made in commit (E) on the `feature` branch, without merging all the other unfinished work.
```text
 (A) --- (B) --- (C)  <-- [main] <-- [HEAD]
          \
          (D) --- (E) --- (F)  <-- [feature]
```

**Action:** `git cherry-pick E`

**Resulting Graph:**
Git copies the changes from (E) and creates a brand new commit (E') on `main`. It has a different hash because it has a different parent.

```text
 (A) --- (B) --- (C) --- (E')  <-- [main] <-- [HEAD]
          \
          (D) --- (E) --- (F)  <-- [feature]
```

#### Interactive Rebase Graph (Squashing)
Interactive rebase (`git rebase -i`) allows you to pause time and rewrite a series of commits before they are finalized. It is commonly used to "squash" multiple messy commits into one.

**Initial State:** You have finished a feature, but you made 3 messy commits (`wip`, `fix typo`, `actually blue`).
```text
 (A) --- (B) (main)
          \
          (C) --- (D) --- (E)  <-- [feature] <-- [HEAD]
          ^       ^       ^
        "wip"  "typo"  "done"
```

**Action:** `git rebase -i main` (We choose to `squash` D and E into C).

**Resulting Graph:**
Git melts commits C, D, and E down and creates a single, clean commit (F).

```text
 (A) --- (B) (main)
          \
          (F)  <-- [feature] <-- [HEAD]
          ^
    "feat: add button"
```

# Common use cases for rebasing

## 1. Keep your feature branch up to date with `main`

### Situation

You started a feature branch a few days ago:

```bash
main:     A---B---C
               \
feature:        D---E
```

Meanwhile, `main` moved forward:

```bash
main:     A---B---C---F---G
               \
feature:        D---E
```

### What you do

```bash
git checkout feature
git rebase main
```

### Result

```bash
main:     A---B---C---F---G
                           \
feature:                    D'---E'
```

### Why rebase here?

* Keeps history **linear**
* Avoids a messy merge commit
* Makes it look like you started from the latest code

Use this when:

* You're working alone on the branch
* You want clean history before merging

---

## 2. Clean up messy commits before merging (interactive rebase)

### Situation

Your commits look like this:

```bash
D - "fix"
E - "oops"
F - "final fix"
```

### What you do

```bash
git rebase -i main
```

Then squash:

```
pick D
squash E
squash F
```

### Result

```bash
D' - "Add login feature"
```

### Why rebase here?

* Turns messy work into **clean, meaningful commits**
* Makes code review easier

Use this when:

* You're about to open a PR
* You want professional-looking history

---

### Why rebase here?

* Keeps dependency chain intact
* Avoids weird merge commits between features
Use this when:

* You have layered branches (very common in big features)

---

## 3. Before merging into `main` (linear history teams)

Some teams prefer:

* **Rebase + fast-forward merge**
* No merge commits at all

### Workflow

```bash
git checkout feature
git rebase main
git checkout main
git merge feature  # fast-forward
```

### Result

```bash
main: A---B---C---D'---E'
```

***

Git is powerful, and with the right mental model—seeing it as a graph of snapshots governed by moving pointers—you can navigate any version control crisis with confidence.

***
----
# Reference
- [The Missing Semester of Your CS Education](https://missing.csail.mit.edu/2020/version-control/)
- [Learngitbranching.js.org](https://learngitbranching.js.org/)