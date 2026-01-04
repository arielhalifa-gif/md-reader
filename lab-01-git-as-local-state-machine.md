# Lab 1 — Git as a Local State Machine
## Understanding Where Your Code Is and Why

---

## 0. Lab Introduction

Welcome to your first Git lab.

This lab is not about memorizing commands. It is about building a mental model of how Git works as a system. If you understand the model, the commands will make sense. If you skip the model and memorize commands, you will be confused every time something unexpected happens.

### What This Lab Is About

You will learn:
- That Git is a **local system** running on your machine
- That your files move through **distinct states**
- That Git does **not automatically save** your changes
- That Git tracks **snapshots**, not live files

By the end of this lab, you should be able to answer confidently:  
**"Where is my code right now, and how did it get there?"**

### What This Lab Is NOT About

This lab does not cover:
- GitHub, GitLab, or any remote hosting service
- Branches
- Merging or rebasing
- Collaboration workflows
- Advanced Git features

Those topics come later. Right now, we focus on the foundation.

### Expectations

- This lab will feel slow. That is intentional.
- You will run the same inspection commands repeatedly. That is intentional.
- You will be asked to predict outcomes before running commands. That is intentional.
- You will make mistakes. That is expected and safe.

Take your time. Observe carefully. Build the mental model.

---

## 1. Mental Model (Before Touching the Keyboard)

Before you run a single Git command, you need to understand what Git is doing conceptually.

### Git Is a Three-State System

When you work with Git, your files exist in one of three states:

1. **Working Directory** — This is your normal file system. The files you edit in your text editor. Changes here are **not tracked** by Git until you explicitly tell Git to track them.

2. **Staging Area (Index)** — This is a preparation area. You place files here when you want to include them in your next snapshot. Think of it as a "loading dock" where you gather changes before committing them.

3. **Repository (Commits)** — This is Git's permanent storage. When you commit, Git takes everything in the staging area and saves it as a snapshot. This snapshot is permanent and identified by a unique hash.

### The Flow

Here is how files move through the system:

```
┌─────────────────────┐
│  Working Directory  │  ← You edit files here
│   (your files)      │
└──────────┬──────────┘
           │
           │  git add
           ▼
┌─────────────────────┐
│   Staging Area      │  ← You prepare changes here
│     (index)         │
└──────────┬──────────┘
           │
           │  git commit
           ▼
┌─────────────────────┐
│    Repository       │  ← Git stores snapshots here
│    (commits)        │
└─────────────────────┘
```

### Critical Insight: Git Tracks Snapshots, Not Files

Git does not continuously watch your files. It does not auto-save. When you modify a file, Git does not know about it until you run `git add`. When you stage a file, Git does not commit it until you run `git commit`.

Each commit is a **snapshot** of the staging area at the moment you committed. If you modify a file after staging it, that new change is **not** in the staging area. You must stage it again.

This is the most common source of confusion for beginners. Keep this in mind as you work through the lab.

---

## 2. Observability Toolkit (How We Inspect Git)

You cannot see Git's internal state with your eyes. You must use Git's inspection commands. These commands are your primary tools for understanding what Git is doing.

### The Four Essential Inspection Commands

#### `git status`

**What it shows:**  
A summary of the current state of your working directory and staging area.

**When to use it:**  
After every action. This is your most important command.

**What to look for:**
- Which files are modified but not staged
- Which files are staged and ready to commit
- Which files are untracked (Git doesn't know about them yet)

---

#### `git diff`

**What it shows:**  
The differences between your working directory and the staging area.

**When to use it:**  
When you want to see what changes you've made that are **not yet staged**.

**What to look for:**
- Lines you added (prefixed with `+`)
- Lines you removed (prefixed with `-`)

---

#### `git diff --staged`

**What it shows:**  
The differences between the staging area and the last commit.

**When to use it:**  
When you want to see what changes are **staged and ready to commit**.

**What to look for:**
- What will be included in your next commit

---

#### `git log --oneline --decorate`

**What it shows:**  
A list of commits in your repository, showing their hash and commit message.

**When to use it:**  
When you want to see the history of snapshots.

**What to look for:**
- The commit hash (a unique identifier)
- The commit message
- The `HEAD` pointer (shows where you currently are)

---

### The Inspection Habit

From this point forward, you will develop a habit:

**After every Git command, run `git status`.**

This is not optional. This is how you learn. You will see exactly what changed and why.

---

## 3. Fully Guided Section — Observe Every State Change

Now you will perform a series of small actions and observe Git's response at every step. Read each step carefully. Do not skip ahead.

### Step 1: Create a Working Directory

Open your terminal and navigate to a location where you want to work. Then create a new directory for this lab:

```bash
mkdir git-lab-01
cd git-lab-01
```

**What just happened:**  
You created a normal directory. Git is not involved yet. This is just your file system.

---

### Step 2: Initialize a Git Repository

Run this command:

```bash
git init
```

**Expected output:**
```
Initialized empty Git repository in /path/to/git-lab-01/.git/
```

**What just happened:**  
Git created a hidden `.git` directory inside `git-lab-01`. This directory contains all of Git's internal data. Your directory is now a Git repository.

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

**What this means:**
- You are on a branch called `master` (ignore branches for now)
- You have no commits yet (the repository is empty)
- There is nothing to commit (no files exist)

**Reflection question:**  
Why does Git say "nothing to commit" even though you just initialized the repository?

**Answer:**  
Because you haven't created any files yet. Git has nothing to track.

---

### Step 3: Create a File

Create a file called `notes.txt` with some content:

```bash
echo "Git is a local system" > notes.txt
```

**What just happened:**  
You created a file in your working directory. This file exists on your file system, but Git does not know about it yet.

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	notes.txt

nothing added to commit but untracked files present (use "git add" to track)
```

**What this means:**
- Git sees the file `notes.txt`
- Git calls it "untracked" because you haven't told Git to track it
- Git suggests using `git add` to track it

**Reflection question:**  
Is `notes.txt` in the staging area?

**Answer:**  
No. It is in the working directory only. Git sees it but is not tracking it.

---

### Step 4: Stage the File

Run this command:

```bash
git add notes.txt
```

**What just happened:**  
You told Git to track `notes.txt` and place it in the staging area. Git took a snapshot of the file's current content and stored it in the staging area.

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   notes.txt
```

**What this means:**
- `notes.txt` is now in the staging area
- Git calls it "changes to be committed"
- If you commit now, this file will be included in the snapshot

**Now inspect what is staged:**

```bash
git diff --staged
```

**Expected output:**
```diff
diff --git a/notes.txt b/notes.txt
new file mode 100644
index 0000000..a1b2c3d
--- /dev/null
+++ b/notes.txt
@@ -0,0 +1 @@
+Git is a local system
```

**What this means:**
- Git shows you the content that will be committed
- The `+` symbol means this line is being added
- This is what your next commit will contain

**Reflection question:**  
If you modify `notes.txt` right now, will that change be in the staging area?

**Answer:**  
No. The staging area contains a snapshot of `notes.txt` from the moment you ran `git add`. If you modify the file now, the new change will be in the working directory only.

---

### Step 5: Modify the File (After Staging)

Add a second line to `notes.txt`:

```bash
echo "Git tracks snapshots, not files" >> notes.txt
```

**What just happened:**  
You modified `notes.txt` in your working directory. The file now has two lines. But remember: the staging area contains a snapshot from Step 4, which had only one line.

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   notes.txt

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   notes.txt
```

**What this means:**
- Git shows `notes.txt` in **two places**
- "Changes to be committed" — the staging area contains the first version (one line)
- "Changes not staged for commit" — the working directory contains a newer version (two lines)

**This is critical. Read it again.**

**Now inspect the unstaged changes:**

```bash
git diff
```

**Expected output:**
```diff
diff --git a/notes.txt b/notes.txt
index a1b2c3d..e4f5g6h 100644
--- a/notes.txt
+++ b/notes.txt
@@ -1 +1,2 @@
 Git is a local system
+Git tracks snapshots, not files
```

**What this means:**
- `git diff` shows the difference between the working directory and the staging area
- The second line is new and not yet staged

**Now inspect the staged changes:**

```bash
git diff --staged
```

**Expected output:**
```diff
diff --git a/notes.txt b/notes.txt
new file mode 100644
index 0000000..a1b2c3d
--- /dev/null
+++ b/notes.txt
@@ -0,0 +1 @@
+Git is a local system
```

**What this means:**
- `git diff --staged` shows what will be committed
- Only the first line is staged

**Reflection question:**  
If you commit right now, how many lines will be in the committed version of `notes.txt`?

**Answer:**  
One line. The commit will contain only what is in the staging area.

---

### Step 6: Stage the Second Change

Run this command:

```bash
git add notes.txt
```

**What just happened:**  
You updated the staging area with the current content of `notes.txt`. Now the staging area contains both lines.

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   notes.txt
```

**What this means:**
- `notes.txt` is staged and ready to commit
- There are no unstaged changes

**Now inspect the staged changes:**

```bash
git diff --staged
```

**Expected output:**
```diff
diff --git a/notes.txt b/notes.txt
new file mode 100644
index 0000000..e4f5g6h
--- /dev/null
+++ b/notes.txt
@@ -0,0 +1,2 @@
+Git is a local system
+Git tracks snapshots, not files
```

**What this means:**
- Both lines are now staged
- This is what your commit will contain

---

### Step 7: Commit the Changes

Run this command:

```bash
git commit -m "Add initial notes about Git"
```

**What just happened:**  
Git took everything in the staging area and saved it as a permanent snapshot. This snapshot is called a commit. Git assigned it a unique hash.

**Expected output:**
```
[master (root-commit) a1b2c3d] Add initial notes about Git
 1 file changed, 2 insertions(+)
 create mode 100644 notes.txt
```

**What this means:**
- Git created a commit with hash `a1b2c3d` (yours will be different)
- The commit contains 1 file with 2 lines added

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master
nothing to commit, working tree clean
```

**What this means:**
- The working directory matches the staging area
- The staging area matches the last commit
- Everything is synchronized

**Now inspect the commit history:**

```bash
git log --oneline --decorate
```

**Expected output:**
```
a1b2c3d (HEAD -> master) Add initial notes about Git
```

**What this means:**
- You have one commit with hash `a1b2c3d`
- `HEAD` points to this commit (you are currently here)
- The commit message is "Add initial notes about Git"

**Reflection question:**  
Where is your code now?

**Answer:**  
Your code exists in three places:
1. Working directory — the file `notes.txt` with two lines
2. Staging area — a snapshot matching the working directory
3. Repository — a commit containing the same snapshot

All three are synchronized.

---

### Step 8: Modify and Observe Again

Add a third line to `notes.txt`:

```bash
echo "Git does not auto-save" >> notes.txt
```

**Now inspect the state:**

```bash
git status
```

**Expected output:**
```
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   notes.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

**What this means:**
- The working directory has changed
- The staging area and repository have not changed
- You must stage the change if you want to commit it

**Reflection question:**  
If you run `git commit` right now without staging, what will happen?

**Answer:**  
Git will refuse to commit. There is nothing in the staging area to commit. You must run `git add` first.

---

### Summary of the Fully Guided Section

You have now observed the complete cycle:

1. Create a file → working directory
2. Stage the file → staging area
3. Commit the file → repository
4. Modify the file → working directory (again)

You have seen that:
- Git does not automatically track changes
- The staging area is a separate state
- You must explicitly move changes through the states
- Inspection commands show you exactly where your code is

---

## 4. Semi-Guided Section — Predict Before Acting

In this section, you will be given goals, not commands. Before you act, you must write down your prediction. Then you will act and compare your prediction to reality.

### Goal 1: Make Git Show a Modified But Unstaged File

**Your task:**  
Modify `notes.txt` so that `git status` shows it as "modified" but not staged.

**Before you act, write down:**
- What command(s) will you run?
- What will `git status` show?

**Now act:**
- Modify the file
- Run `git status`
- Compare your prediction to the output

**Reflection:**  
Did your prediction match? If not, why?

---

### Goal 2: Make Git Show a Staged File

**Your task:**  
Stage the changes you just made so that `git status` shows them as "changes to be committed."

**Before you act, write down:**
- What command will you run?
- What will `git status` show?
- What will `git diff` show?
- What will `git diff --staged` show?

**Now act:**
- Stage the file
- Run all three inspection commands
- Compare your predictions to the output

**Reflection:**  
Did your predictions match? If not, why?

---

### Goal 3: Return to a Clean Working Tree

**Your task:**  
Commit the staged changes so that `git status` shows "nothing to commit, working tree clean."

**Before you act, write down:**
- What command will you run?
- What will `git status` show after the commit?
- What will `git log --oneline --decorate` show?

**Now act:**
- Commit the changes with a meaningful message
- Run the inspection commands
- Compare your predictions to the output

**Reflection:**  
Did your predictions match? If not, why?

---

### Goal 4: Create a Situation Where the Same File Appears in Two States

**Your task:**  
Make `git status` show `notes.txt` in both "changes to be committed" and "changes not staged for commit" at the same time.

**Before you act, write down:**
- What sequence of actions will create this situation?
- Why will Git show the file in both places?

**Now act:**
- Perform the actions
- Run `git status`
- Run `git diff`
- Run `git diff --staged`
- Compare your predictions to the output

**Reflection:**  
This is the most confusing state for beginners. Do you understand why Git shows the file in two places?

---

## 5. Intermediate Debug Section — Fix a Real Mistake

You will now encounter a realistic mistake and learn how to inspect and fix it.

### The Scenario

You have been working on `notes.txt`. You made some changes, staged them, and committed them. But after committing, you realized you committed something you didn't want.

**Set up the scenario:**

1. Add a line to `notes.txt`:
   ```bash
   echo "This is a mistake" >> notes.txt
   ```

2. Stage and commit it:
   ```bash
   git add notes.txt
   git commit -m "Add a line (oops, this was a mistake)"
   ```

3. Now inspect the state:
   ```bash
   git status
   git log --oneline --decorate
   ```

**The problem:**  
You committed "This is a mistake" but you didn't want to. You want to remove that line from the file and from the commit history.

**Constraints:**
- Do not delete the repository
- Do not start over
- Do not use advanced history-rewriting commands (you don't know them yet)

---

### Your Task: Inspect and Reason

Before you fix anything, answer these questions:

1. **Where is the mistake right now?**
   - Is it in the working directory?
   - Is it in the staging area?
   - Is it in the repository?

2. **What do you want the final state to be?**
   - What should `notes.txt` contain?
   - What should the commit history look like?

3. **What are your options?**
   - Can you edit the file and commit again?
   - Can you undo the last commit?
   - What are the trade-offs?

---

### Guided Fix: Create a New Commit

The safest and simplest fix is to create a new commit that removes the mistake.

**Step 1: Remove the line from the file**

Open `notes.txt` in your text editor and delete the line "This is a mistake". Save the file.

**Step 2: Inspect the state**

```bash
git status
```

**Expected output:**
```
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   notes.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

**What this means:**
- You modified the file in the working directory
- The change is not staged yet

**Step 3: Inspect the change**

```bash
git diff
```

**Expected output:**
```diff
diff --git a/notes.txt b/notes.txt
index e4f5g6h..a1b2c3d 100644
--- a/notes.txt
+++ b/notes.txt
@@ -1,3 +1,2 @@
 Git is a local system
 Git tracks snapshots, not files
-This is a mistake
```

**What this means:**
- The line "This is a mistake" is being removed (note the `-` symbol)

**Step 4: Stage and commit the fix**

```bash
git add notes.txt
git commit -m "Remove incorrect line"
```

**Step 5: Inspect the history**

```bash
git log --oneline --decorate
```

**Expected output:**
```
b2c3d4e (HEAD -> master) Remove incorrect line
a1b2c3d Add a line (oops, this was a mistake)
e4f5g6h Add initial notes about Git
```

**What this means:**
- You now have three commits
- The second commit contains the mistake
- The third commit fixes the mistake
- The current state of `notes.txt` is correct

---

### Reflection: Why This Approach?

You might wonder: "Why didn't we just delete the bad commit?"

**Answer:**  
Deleting commits is possible but dangerous, especially if you've shared your work with others. Creating a new commit that fixes the mistake is:
- Safe
- Transparent (the history shows what happened)
- Reversible (you can always go back)

As you learn more advanced Git techniques, you will learn when and how to rewrite history. For now, this approach is correct.

---

## 6. Exit Criteria (What the Student Must Be Able to Explain)

Before you finish this lab, you must be able to explain the following statements clearly and confidently. If you cannot, revisit the relevant section.

### Core Concepts

- [ ] **I can explain what the working directory is.**
  - It is the normal file system where I edit files. Changes here are not tracked by Git until I stage them.

- [ ] **I can explain what the staging area is.**
  - It is a preparation area where I gather changes before committing. It contains a snapshot of files at the moment I ran `git add`.

- [ ] **I can explain what the repository is.**
  - It is Git's permanent storage. When I commit, Git saves a snapshot of the staging area as a commit.

- [ ] **I can explain why Git does not auto-save.**
  - Git tracks snapshots, not live files. I must explicitly stage and commit changes.

---

### Inspection Skills

- [ ] **I can use `git status` to see the current state.**
  - I know it shows untracked files, unstaged changes, and staged changes.

- [ ] **I can use `git diff` to see unstaged changes.**
  - I know it shows the difference between the working directory and the staging area.

- [ ] **I can use `git diff --staged` to see staged changes.**
  - I know it shows the difference between the staging area and the last commit.

- [ ] **I can use `git log --oneline --decorate` to see commit history.**
  - I know it shows commit hashes, messages, and the `HEAD` pointer.

---

### State Transitions

- [ ] **I can explain how a file moves from the working directory to the staging area.**
  - I run `git add <file>`.

- [ ] **I can explain how a file moves from the staging area to the repository.**
  - I run `git commit -m "message"`.

- [ ] **I can explain why a file might appear in two states at once.**
  - If I stage a file, then modify it again, the staging area contains the old version and the working directory contains the new version.

---

### Debugging

- [ ] **I can explain why a file is not included in a commit.**
  - Either it was not staged, or it was modified after staging.

- [ ] **I can explain how to inspect Git instead of guessing.**
  - I run `git status`, `git diff`, and `git diff --staged` after every action.

- [ ] **I can explain how to fix a committed mistake.**
  - I create a new commit that corrects the mistake, rather than deleting the bad commit.

---

## Conclusion

You have completed Lab 1.

You now understand that Git is a local state machine. Your files move through distinct states, and Git does not automatically save changes. You have learned to inspect Git's state at every step and to reason about where your code is and why.

This mental model is the foundation for everything else you will learn about Git. Branches, merging, remotes, and collaboration all build on this foundation.

In the next lab, you will learn how to navigate through commit history and restore previous states.

**Before you move on:**
- Review the exit criteria
- Make sure you can explain each concept
- Practice the inspection commands until they become automatic

Git is a tool for managing state. You now understand the states. Everything else is just operations on those states.

Well done.
