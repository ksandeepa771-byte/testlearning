# Week 01 Monday March 30, 2026 - Linux Basics Failure 01

## Case

This was my first debugger-first lab. I built a small Bash script in `/opt/devops-demo`, made it work, then broke it on purpose in two ways. First I removed execute permission. Second I renamed the script so the expected file path no longer existed. The goal was to learn how permissions, file names, and first-check commands help me debug simple Linux problems without guessing.

## What I built first

I connected to the Ubuntu lab server, created `/opt/devops-demo` and `logs`, changed ownership to the `ubuntu` user, and created `hello.sh`.

The script writes one line to the screen and also appends it to the log file. The command `'printf ... > hello.sh'` means: print the script text exactly as written and write it into a file called `hello.sh`. The `'\n'` parts mean new lines. The `'>'` symbol means overwrite the file with new content.

The first script line, `#!/usr/bin/env bash`, tells Linux to run the script with Bash. Bash is a Linux shell, which means it is one program used to read and run typed commands.

Inside the second line, `'echo "hello from devops demo"'` means print text. `'tee -a /opt/devops-demo/logs/hello.log'` means show that same text on screen and also append it to the log file. The `'-a'` flag means append, so old log content is kept.

I also used `'sudo chown -R ubuntu:ubuntu /opt/devops-demo'`. This was needed because `/opt` is a system path and is often owned by `root`. The `'-R'` flag means recursive, so ownership changed for the folder and everything inside it. In a normal home folder like `'~'`, I usually do not need to do this because those files are already owned by me.

To run the script directly as `'./hello.sh'`, I needed `'chmod +x hello.sh'`. The `'+x'` adds execute permission. Without execute permission, the file can still exist and be readable, but it cannot be run directly.

## Clean build proof

The file and folder looked correct after setup. The output of `'ls -l /opt/devops-demo'` showed `hello.sh` owned by `ubuntu` and the `logs` directory also owned by `ubuntu`. The `'x'` in the permission string proved the script had execute permission. The command `'file hello.sh'` reported that it was a `Bourne-Again shell script, ASCII text executable`, which confirmed it was a Bash script and runnable. Then `'./hello.sh'` worked, `'cat /opt/devops-demo/logs/hello.log'` showed the log line, and `'echo $?'` returned `0`, which means success.

## Deliberate break 01: remove execute permission

I ran `'chmod 644 hello.sh'`. This removed execute permission. After that, `'ls -l hello.sh'` showed a permission string like `-rw-r--r--`, so there was no `x` anymore. When I tried `'./hello.sh'`, Linux returned `Permission denied`. Then `'echo $?'` returned `126`. This taught me that exit code `126` means the file exists, but it cannot be executed.

## First checks for break 01

I used the schedule checks: `'pwd'`, `'ls -l /opt/devops-demo'`, `'file /opt/devops-demo/*'`, and `'echo $?'`.

These checks told me:
- I was in the correct directory
- the file still existed
- the file was still a Bash script
- the previous check itself succeeded

Then I ran `'bash hello.sh'` and it worked. That showed the script content was fine. The real issue was permission. This helped me understand an important difference:
- `'./hello.sh'` needs execute permission
- `'bash hello.sh'` works because Bash reads the file as input

## Fix for break 01

I restored execute permission with `'chmod +x hello.sh'`. After that, `'ls -l hello.sh'` showed `x` again, `'./hello.sh'` worked, and `'echo $?'` returned `0`.

## Deliberate break 02: rename the script

I then ran `'mv hello.sh hello-old.sh'`. The file still existed, but now it had the wrong name. When I tried `'./hello.sh'`, Linux returned `No such file or directory`. Then `'echo $?'` returned `127`. This taught me that exit code `127` usually means the command or file path was not found.

## First checks for break 02

I checked `'pwd'`, `'ls -l'`, and `'file *'`.

These checks proved that:
- I was still in `/opt/devops-demo`
- the script now existed as `hello-old.sh`
- the file itself was still a valid Bash script

So the content was not broken. The real problem was the file name and path.

## Fix for break 02

I renamed the script back with `'mv hello-old.sh hello.sh'`. After that, `'ls -l'` showed the expected name again, `'./hello.sh'` worked, `'cat logs/hello.log'` showed multiple successful runs, and `'echo $?'` returned `0`.

## Path confusion I learned

I tested `'cat /logs/hello.log'` and it failed, but `'cat /opt/devops-demo/logs/hello.log'` and `'cat logs/hello.log'` worked.

This taught me:
- `'/logs/hello.log'` is an absolute path starting from the Linux root `/`
- `'/opt/devops-demo/logs/hello.log'` is the real absolute path
- `'logs/hello.log'` is a relative path starting from my current folder `/opt/devops-demo`

So a path starting with `'/'` always starts from the root of the filesystem, not from the folder I am currently in.

## Evidence that mattered most

The most useful evidence was `'ls -l'` for permissions and file names, `'file'` for confirming the script type, `'echo $?'` for exit code meaning, and the direct comparison between `'./hello.sh'` and `'bash hello.sh'`.

## Root cause, fix, and prevention

For break 01, the root cause was removing execute permission with `'chmod 644 hello.sh'`. The fix was `'chmod +x hello.sh'`. To prevent this type of confusion, I should always check `'ls -l'` before assuming the script itself is broken.

For break 02, the root cause was renaming the file to `hello-old.sh`, so the expected path `./hello.sh` no longer existed. The fix was `'mv hello-old.sh hello.sh'`. To prevent this, I should check file names and paths with `'ls -l'` and be careful when renaming files.

## What I guessed wrong at first

I typed `'pdw'` instead of `'pwd'`. I also checked `'/logs/hello.log'` even though the real path was under `'/opt/devops-demo/logs/hello.log'`. I briefly tested the wrong name `hello.ssh` instead of `hello.sh`.

## Final Commands I Ran

- `'ssh -i ~/Downloads/tg-key-tokyo.pem ubuntu@43.207.140.32'` -> `'-i'` means use this private key file for SSH
- `'sudo mkdir -p /opt/devops-demo/logs'` -> `'-p'` creates parent folders if needed
- `'sudo chown -R ubuntu:ubuntu /opt/devops-demo'` -> `'-R'` means recursive owner change
- `'cd /opt/devops-demo'`
- `'printf "#!/usr/bin/env bash\necho \"hello from devops demo\" | tee -a /opt/devops-demo/logs/hello.log\n" > hello.sh'` -> `'\n'` means new line and `'>'` writes into the file
- `'chmod +x hello.sh'` -> `'+x'` adds execute permission
- `'ls -l /opt/devops-demo'`
- `'file hello.sh'`
- `'./hello.sh'`
- `'cat /opt/devops-demo/logs/hello.log'`
- `'echo $?'` -> `'$?'` shows the exit code of the previous command
- `'chmod 644 hello.sh'`
- `'ls -l hello.sh'`
- `'./hello.sh'`
- `'echo $?'`
- `'pwd'`
- `'ls -l /opt/devops-demo'`
- `'file /opt/devops-demo/*'`
- `'bash hello.sh'`
- `'chmod +x hello.sh'`
- `'ls -l hello.sh'`
- `'./hello.sh'`
- `'echo $?'`
- `'mv hello.sh hello-old.sh'`
- `'ls -l'`
- `'./hello.sh'`
- `'echo $?'`
- `'pwd'`
- `'file *'`
- `'mv hello-old.sh hello.sh'`
- `'ls -l'`
- `'./hello.sh'`
- `'cat logs/hello.log'`
- `'echo $?'`
