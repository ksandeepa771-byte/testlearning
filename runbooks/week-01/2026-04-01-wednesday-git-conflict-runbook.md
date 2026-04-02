# Week 01 Wednesday April 1, 2026 - Git Conflict Runbook

## Case

This lab was about two common Git problems. First I broke the remote on purpose so `git push` would fail. After that I created a real merge conflict by changing the same `README.md` line on `main` and `debug-lab`. The goal was to stop guessing and use Git evidence commands to see what actually happened.

## Clean build first

I started from the capstone repo in `'D:/aws-devops-capstone'` and checked the clean state with `'git status'` and `'git remote -v'`.

I then created a simple clean base:

- `'mkdir -p docs notes runbooks/week-01'` made sure the main folders existed
- `'printf ... > README.md'` created a fresh `README.md`
- `'printf ... > notes/week-01-overview.md'` created a simple Week 01 notes file
- `'git add README.md notes/week-01-overview.md'` staged the clean files
- `'git commit -m "Set up capstone README and notes structure"'` created the clean baseline commit

The `'-m'` flag means commit message.

## Deliberate break 01: wrong remote URL

I checked the real remote first with `'git remote -v'`, then changed it with:

- `'git remote set-url origin https://github.com/ksandeepa771-byte/testlearning-wrong.git'`

This was the deliberate break. It changed the `origin` remote to a bad URL so a push should fail.

The command `'git remote -v'` was the first important check because it shows the fetch and push URLs exactly. When the remote is wrong, Git push problems can be caused by the destination itself, not by my branch or commit history.

Then I fixed it by restoring:

- `'git remote set-url origin https://github.com/ksandeepa771-byte/testlearning.git'`

After that, pushing worked again. This taught me to check `'git remote -v'` early any time a push fails.

## Deliberate break 02: real merge conflict

I created a new branch with `'git switch -c debug-lab'`. The `'-c'` means create the branch and switch to it in one step.

On `debug-lab` I changed the same `README.md` goal line to one version and committed it with:

- `'git commit -m "Change README goal on debug-lab branch"'`

Then I switched back to `main`, changed that exact same line differently, and committed it with:

- `'git commit -m "Change README goal on main branch"'`

That created the setup for a real merge conflict because both branches changed the same line in different ways.

## Expected symptom

When I ran `'git merge debug-lab'`, Git stopped and reported:

- `CONFLICT (content): Merge conflict in README.md`
- `Automatic merge failed; fix conflicts and then commit the result.`

This was the expected symptom. Git could not decide which version of the same line should win.

## Evidence that mattered

The most useful commands were the exact first checks from the schedule:

- `'git status'`
- `'git log --oneline --graph --all --max-count=10'`
- `'git diff'`

`'git status'` showed:

- I was on `main`
- the repo was in a merge state shown as `'(main|MERGING)'`
- there was an unmerged path
- `README.md` was listed as `both modified`

`'git log --oneline --graph --all --max-count=10'` showed the branch split clearly:

- `main` had its own README commit
- `debug-lab` had its own README commit
- both came from the same earlier base commit

`'git diff'` showed the real conflict markers:

- `'<<<<<<< HEAD'` means the current branch version, which was `main`
- `'======='` separates the two versions
- `'>>>>>>> debug-lab'` means the incoming branch version from `debug-lab`

This taught me that conflict markers are not random text. They are Git showing both competing versions of the same change.

## Why the merge conflict happened

The root cause was simple: I changed the same `README.md` line in two branches and then tried to merge them. Git could not auto-merge that exact line because both branches gave it different content for the same location.

## Fix

I opened the conflicted `README.md`, removed the conflict markers, and wrote one final combined line:

- `'Build a debugger-first AWS DevOps capstone repo with notes, runbooks, and Git practice.'`

Then I finished the resolution with:

- `'git add README.md'`
- `'git commit -m "Resolve README merge conflict"'`

After that, `'git status'` showed a clean working tree again.

## What `ahead of origin/main by 3 commits` meant

After the conflict was resolved, Git said:

- `'Your branch is ahead of 'origin/main' by 3 commits.'`

This meant my local `main` had 3 commits that GitHub did not have yet. In this case they were:

- the `main` branch README change
- the `debug-lab` commit that became part of `main` through the merge
- the merge resolution commit

It did not mean there were uncommitted file changes. The line `'nothing to commit, working tree clean'` only means the working tree is clean. Push state is a different thing.

## What I guessed wrong at first

I first typed:

- `'git swtich -c debug-lab'`
- `'git switich -c debug-lab'`

Both failed because the command name was wrong. The correct command was `'git switch -c debug-lab'`.

That was a useful reminder that Git typos can look like deeper Git problems, but sometimes the issue is just the command spelling.

## Prevention

To avoid confusion next time:

- check `'git remote -v'` before blaming GitHub or authentication
- check `'git status'` first during merges
- use `'git log --oneline --graph --all'` to understand branch history before merging
- read conflict markers carefully instead of panicking
- resolve the file deliberately, then `'git add'` and `'git commit'`

## Final Commands I Ran

- `'cd /d/aws-devops-capstone'`
- `'git status'`
- `'git remote -v'`
- `'mkdir -p docs notes runbooks/week-01'` -> `'-p'` creates parent folders if needed
- `'printf ... > README.md'` -> `'>'` overwrites the file
- `'printf ... > notes/week-01-overview.md'`
- `'git add README.md notes/week-01-overview.md'`
- `'git commit -m "Set up capstone README and notes structure"'` -> `'-m'` means commit message
- `'git log --oneline --graph --all --max-count=5'` -> `'--graph'` draws branch history and `'--all'` shows all refs
- `'git remote set-url origin https://github.com/ksandeepa771-byte/testlearning-wrong.git'`
- `'git remote -v'`
- `'git push origin main'`
- `'echo $?'` -> `'$?'` shows the previous command exit code
- `'git remote set-url origin https://github.com/ksandeepa771-byte/testlearning.git'`
- `'git remote -v'`
- `'git push origin main'`
- `'echo $?'`
- `'git switch -c debug-lab'` -> `'-c'` means create and switch
- `'printf ... > README.md'`
- `'git add README.md'`
- `'git commit -m "Change README goal on debug-lab branch"'`
- `'git switch main'`
- `'printf ... > README.md'`
- `'git add README.md'`
- `'git commit -m "Change README goal on main branch"'`
- `'git merge debug-lab'`
- `'git status'`
- `'git log --oneline --graph --all --max-count=10'`
- `'git diff'`
- `'cat README.md'`
- `'printf ... > README.md'`
- `'git add README.md'`
- `'git commit -m "Resolve README merge conflict"'`
- `'git status'`
