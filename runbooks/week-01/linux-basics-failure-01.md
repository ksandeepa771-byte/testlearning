# Week 01 - Linux Basics Failure 01

Date / week / case:
- Schedule day: Monday, March 30, 2026
- Lab run date: April 1, 2026
- Week: 01
- Case: shell script permission and file-name/path debugging in `/opt/devops-demo`

## Goal

Build a tiny working shell script in `/opt/devops-demo`, then break it on purpose and recover it by checking evidence instead of guessing.

## Clean working build

I created `/opt/devops-demo` and `/opt/devops-demo/logs` on the Ubuntu lab server. I changed ownership to `ubuntu:ubuntu` so I could work there without editing everything as root. Then I created `hello.sh`, made it executable, and ran it successfully. The script printed `hello from devops demo` and appended the same message to `/opt/devops-demo/logs/hello.log`.

I also learned a few small but useful points during the build:
- I first forgot to use `> hello.sh` with `printf`, so the script text printed to the terminal instead of being saved into a file.
- `file hello.sh` showed `Bourne-Again shell script, ASCII text executable`, which confirmed that the file was a Bash script and had execute permission.
- `cat /logs/hello.log` failed because `/logs/...` is an absolute path from the root directory, but the real file was at `/opt/devops-demo/logs/hello.log`.

## What I intentionally broke

I broke two things on purpose:

1. I ran `chmod 644 hello.sh`, which removed execute permission from the script.
2. I renamed the file with `mv hello.sh hello-old.sh`, which changed the path and file name without updating what I was trying to run.

## Expected blast radius

Only the tiny demo script should fail. The rest of the server should stay healthy. The expected impact is local to `/opt/devops-demo` and should not affect SSH, system services, or other users.

## Symptoms that appeared

After `chmod 644 hello.sh`, running `./hello.sh` failed with:

```text
-bash: ./hello.sh: Permission denied
```

After renaming the script to `hello-old.sh`, running `./hello.sh` failed with:

```text
-bash: ./hello.sh: No such file or directory
```

I also saw these extra small mistakes during learning:
- `pdw` failed because the correct command is `pwd`
- `file /opt/devops-demo/hello.ssh` failed because the real file name was `hello.sh`
- `cat /logs/hello.log` failed because the real file path was `/opt/devops-demo/logs/hello.log`

## What evidence I checked first

The first commands I used were:

```text
pwd
ls -l /opt/devops-demo
file /opt/devops-demo/*
echo $?
```

These checks were enough to tell me:
- where I was
- whether the file existed
- whether the file was a script or directory
- whether the previous command succeeded or failed

## What evidence actually mattered

The most useful evidence was:

```text
ls -l hello.sh
```

This showed:

```text
-rw-r--r-- 1 ubuntu ubuntu 91 Apr  1 10:50 hello.sh
```

That proved the file still existed but did not have execute permission because there was no `x` in the permission string.

This command also mattered:

```text
bash hello.sh
```

It still worked, which proved that the script content itself was fine and the problem was execute permission, not bad script logic.

For the rename problem, this output mattered most:

```text
ls -l
```

It showed:

```text
hello-old.sh
logs
```

That proved the file had simply been renamed, so `./hello.sh` was looking for the wrong name.

## Root cause

The first failure happened because direct execution with `./hello.sh` requires execute permission, and `chmod 644 hello.sh` removed that execute bit.

The second failure happened because the file no longer existed under the name `hello.sh`. It had been renamed to `hello-old.sh`, so the shell could not find `./hello.sh`.

## Fix

I fixed the first problem with:

```text
chmod +x hello.sh
```

Then I confirmed it worked again with:

```text
./hello.sh
echo $?
```

I fixed the second problem with:

```text
mv hello-old.sh hello.sh
```

Then I ran the script again and checked the log file:

```text
./hello.sh
cat logs/hello.log
echo $?
```

## How to prevent it next time

- After changing permissions, always confirm with `ls -l` before assuming the script is executable.
- If a script fails with `Permission denied`, try `bash hello.sh` to separate script-content problems from execute-permission problems.
- If a file suddenly looks missing, use `pwd`, `ls -l`, and `file *` before guessing.
- Be careful with output redirection. If I forget `> hello.sh`, the script text prints to the terminal and no file is created.
- Use the real absolute path when checking files in system directories like `/opt`.

## Time to detect

I detected both failures immediately because the command output clearly changed from successful execution to either `Permission denied` or `No such file or directory`.

## Time to recover

Recovery was quick once I checked the basic evidence. The permission issue took only one permission fix. The rename issue took one `mv` command to restore the original file name.

## What I guessed wrong at first

- I first typed `pdw` instead of `pwd`
- I checked `/logs/hello.log` instead of `/opt/devops-demo/logs/hello.log`
- I checked `hello.ssh` instead of `hello.sh`
- I first used `printf` without `> hello.sh`, so I printed the script text instead of saving it

## Simple mental model

```text
Script fails
   ->
Check current directory
   ->
Check file name and permissions
   ->
Check file type
   ->
Try direct run or bash run
   ->
Fix permission or path/name problem
```

## Commands I ran

```bash
ssh -i ~/Downloads/tg-key-tokyo.pem ubuntu@43.207.140.32
sudo mkdir -p /opt/devops-demo/logs
sudo chown -R ubuntu:ubuntu /opt/devops-demo
cd /opt/devops-demo
printf '#!/usr/bin/env bash\necho "hello from devops demo" | tee -a /opt/devops-demo/logs/hello.log\n' > hello.sh
chmod +x hello.sh
ls -l /opt/devops-demo
file hello.sh
./hello.sh
cat /opt/devops-demo/logs/hello.log
echo $?
chmod 644 hello.sh
ls -l hello.sh
./hello.sh
echo $?
pwd
ls -l /opt/devops-demo
file /opt/devops-demo/*
bash hello.sh
chmod +x hello.sh
ls -l hello.sh
./hello.sh
echo $?
mv hello.sh hello-old.sh
ls -l
./hello.sh
echo $?
pwd
ls -l
file *
mv hello-old.sh hello.sh
ls -l
./hello.sh
cat logs/hello.log
echo $?
```
