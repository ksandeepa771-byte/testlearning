# Week 01 Tuesday March 31, 2026 - SSH Auth Debug

## Case

This lab was about making SSH easier to use with a local config file, then breaking authentication on purpose and reading the evidence instead of guessing. I created a `Host` alias called `lab-host`, connected through it, tested file transfer with `scp`, then changed the SSH config to point to a wrong key file so authentication would fail. After that I used SSH debug output, file checks, and the config file itself to find the real cause and fix it.

## What I built first

I already had a local SSH folder at `'~/.ssh'`, but I did not have a config file yet. That is normal. The file `'~/.ssh/config'` is optional. EC2 does not automatically create it on my Windows PC. It is a local SSH client config file that I can create myself when I want saved connection settings.

I checked this with `'ls -la ~/.ssh'` and `'test -f ~/.ssh/config && cat ~/.ssh/config'`. The `'test -f'` part checks whether the file exists. When `'echo $?'` returned `1`, that meant the file was not there yet.

## The SSH config I created

I added a host entry for one lab server:

- `'Host lab-host'` means a nickname or alias that I choose myself
- `'HostName 43.207.140.32'` means the real server IP address
- `'User ubuntu'` means the remote Linux username
- `'IdentityFile ~/Downloads/tg-key-tokyo.pem'` means which private key file to use
- `'IdentitiesOnly yes'` means SSH should try only that chosen key, not many other saved keys

This taught me an important difference:

- `Host` is my local shortcut name
- `HostName` is the real destination address

So `'ssh lab-host'` works like a shorter version of `'ssh -i ~/Downloads/tg-key-tokyo.pem ubuntu@43.207.140.32'`.

## Clean build proof

After creating the config, `'cat ~/.ssh/config'` showed the saved settings clearly. Then `'ssh lab-host "whoami; hostname; pwd"'` worked.

That command proved:

- `'whoami'` showed I logged in as `ubuntu`
- `'hostname'` showed the remote machine name `ip-172-31-40-203`
- `'pwd'` showed my remote working directory `/home/ubuntu`

The semicolons `';'` mean run the remote commands one after another.

## File transfer check

I created one local test file with `'printf "ssh config practice\n" > ssh-config-test.txt'`, uploaded it with `'scp ssh-config-test.txt lab-host:~/'`, checked it remotely with `'ssh lab-host "cat ~/ssh-config-test.txt"'`, and downloaded it back with `'scp lab-host:~/ssh-config-test.txt ./ssh-config-test-from-server.txt'`.

Then I compared the original and downloaded copies with `'diff ssh-config-test-from-server.txt ssh-config-test.txt'`. There was no output, and `'echo $?'` returned `0`, which means both files matched.

## What `ssh-keygen -F 43.207.140.32` means

This command does not show the public key of my private key file. It searches my local `known_hosts` data for saved host keys for that server IP.

The output lines like:

- `'43.207.140.32 ssh-ed25519 ...'`
- `'43.207.140.32 ssh-rsa ...'`
- `'43.207.140.32 ecdsa-sha2-nistp256 ...'`

are server host public keys. They belong to the server, not to my `.pem` file. This is part of server identity verification.

So I learned there are two different key ideas in SSH:

- my private key and matching public key are for proving my identity to the server
- the server host public key is for proving the server's identity to me

That is why `'ssh-keygen -F 43.207.140.32'` is about the server I trust, not about my own login key.

## Deliberate break

I first made a safe backup with `'cp ~/.ssh/config ~/.ssh/config.mar31.bak'`. Then I replaced the working config so that `'IdentityFile ~/.ssh/id_ed25519_wrong'` pointed to a key file that did not exist.

This was the deliberate break. The expected symptom was that SSH public-key authentication would fail because the config told SSH to use a missing private key.

## What the debug SSH command means

I ran `'ssh -vvv -o BatchMode=yes -o PreferredAuthentications=publickey lab-host "true"'`.

This command means:

- `'ssh'` starts an SSH connection
- `'-vvv'` shows very detailed debug output
- `'-o BatchMode=yes'` tells SSH not to stop and ask interactive questions
- `'-o PreferredAuthentications=publickey'` tells SSH to test public-key authentication
- `'lab-host'` means use the saved alias from my config
- `'"true"'` is a tiny remote command that does nothing and exits successfully if login works

So this was a clean test: if SSH authentication worked, the remote command would exit immediately. If SSH authentication was broken, the command would fail before it ever ran.

## Evidence that mattered

The most important evidence was in the debug output. These lines mattered most:

- `'Reading configuration data /c/Users/pc/.ssh/config'` showed SSH really read my config file
- `'Applying options for lab-host'` showed it matched the `Host` alias
- `'identity file /c/Users/pc/.ssh/id_ed25519_wrong type -1'` showed the configured key path was invalid
- `'Will attempt key: /c/Users/pc/.ssh/id_ed25519_wrong explicit'` showed SSH was really trying that bad path
- `'no such identity: /c/Users/pc/.ssh/id_ed25519_wrong: No such file or directory'` showed the real failure
- `'Permission denied (publickey).'` showed authentication failed

After that, `'echo $?'` returned `255`, which is a common SSH failure exit code.

The other checks supported the same conclusion:

- `'ls -la ~/.ssh'` showed the wrong key file did not exist
- `'test -f ~/.ssh/id_ed25519_wrong'` returned `1`, which means the file was not found
- `'cat ~/.ssh/config'` showed the broken `IdentityFile` line clearly

## Fix

I restored the correct config so that `'IdentityFile ~/Downloads/tg-key-tokyo.pem'` pointed to the real private key again.

After that:

- `'cat ~/.ssh/config'` showed the corrected config
- `'ssh lab-host "whoami; hostname; pwd"'` worked again
- `'echo $?'` returned `0`

That proved the root cause was the wrong key path in the SSH config file.

## Root cause, fix, and prevention

The root cause was a wrong `IdentityFile` path inside `'~/.ssh/config'`. SSH was not failing because the server was down. It was failing because my local config told SSH to use a private key that did not exist.

The fix was to restore the correct path to `'~/Downloads/tg-key-tokyo.pem'`.

To prevent this in future:

- check `'cat ~/.ssh/config'` early
- use `'ssh -vvv ...'` when authentication is unclear
- confirm the key file exists with `'ls -la ~/.ssh'` or `'test -f <path>'`
- remember that `Host` is only a shortcut name, while `HostName` is the real server address

## What I guessed wrong at first

I first typed a bad here-doc line and pasted terminal output back into Git Bash, which created noise. I also typed `'ssh-keygen -F 43.207.140,32'` with a comma instead of a dot, and I typed `'ssf'` instead of `'ssh'`.

## Final Commands I Ran

- `'ls -la ~/.ssh'`
- `'test -f ~/.ssh/config && cat ~/.ssh/config'` -> `'test -f'` checks whether a regular file exists
- `'echo $?'` -> `'$?'` shows the exit code of the previous command
- `'cat >> ~/.ssh/config <<'EOF' ... EOF'` -> `'>>'` appends text to the file
- `'cat ~/.ssh/config'`
- `'ssh-keygen -F 43.207.140.32'` -> `'-F'` means find this host in known_hosts
- `'ssh lab-host "whoami; hostname; pwd"'`
- `'printf "ssh config practice\n" > ssh-config-test.txt'` -> `'\n'` means new line and `'>'` overwrites the file
- `'scp ssh-config-test.txt lab-host:~/'`
- `'ssh lab-host "cat ~/ssh-config-test.txt"'`
- `'scp lab-host:~/ssh-config-test.txt ./ssh-config-test-from-server.txt'`
- `'diff ssh-config-test-from-server.txt ssh-config-test.txt'`
- `'echo $?'`
- `'cp ~/.ssh/config ~/.ssh/config.mar31.bak'`
- `'cat > ~/.ssh/config <<'EOF' ... EOF'` -> `'>'` overwrites the file with new content
- `'ssh -vvv -o BatchMode=yes -o PreferredAuthentications=publickey lab-host "true"'` -> `'-vvv'` means very verbose debug and `'-o'` passes one SSH option
- `'echo $?'`
- `'ls -la ~/.ssh'`
- `'test -f ~/.ssh/id_ed25519_wrong'`
- `'echo $?'`
- `'cat > ~/.ssh/config <<'EOF' ... EOF'`
- `'cat ~/.ssh/config'`
- `'ssh lab-host "whoami; hostname; pwd"'`
- `'echo $?'`
