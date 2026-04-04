# Week 01 Thursday April 2, 2026 - Service Start Failure

## Case

This lab was about building one tiny `systemd` service cleanly, then breaking its startup path on purpose and debugging the failure with evidence. I used a simple Python HTTP server on port `9090`, managed by a `systemd` unit called `devops-demo.service`. After it worked, I changed the unit file so `ExecStart` pointed to a path that did not exist. The service then failed to start, and I used `systemctl`, `journalctl`, `ls`, and `ss` to prove why.

## What I built first

I created a small web folder under `'/opt/devops-demo/web'` and wanted to write `index.html` there. My first write attempt failed with `Permission denied`. That happened because `/opt/devops-demo` was still owned by `root` from earlier lab work. I fixed that with `'sudo chown -R ubuntu:ubuntu /opt/devops-demo'`, then writing the file worked.

This reminded me of the same Linux ownership rule from the earlier shell lab:

- paths under `'/opt'` are often system-owned
- my normal `ubuntu` user may not be allowed to write there
- `'chown -R'` changes ownership recursively

## The tiny service I created

I created one shell script at `'/usr/local/bin/devops-demo-run'` that changed into the web folder and started:

- `'python3 -m http.server 9090 --bind 0.0.0.0'`

That means:

- the server serves files from `'/opt/devops-demo/web'`
- it listens on port `9090`
- `'0.0.0.0'` means listen on all IPv4 interfaces on that machine

I then created the unit file `'/etc/systemd/system/devops-demo.service'` with:

- `'ExecStart=/usr/local/bin/devops-demo-run'`
- `'Restart=on-failure'`
- `'User=root'`

For this simple lab, the service and the server are related but not identical:

- the **server** is the Python program that answers HTTP requests
- the **service** is the `systemd`-managed background unit that starts and monitors that server

## Clean build proof

The service worked in the clean state.

`'systemctl status devops-demo --no-pager'` showed:

- `Loaded: loaded`
- `Active: active (running)`
- `Main PID: 55232 (python3)`

`'journalctl -u devops-demo -n 20 --no-pager'` showed the service started.

`'ss -ltnp | grep 9090'` showed:

- `LISTEN ... 0.0.0.0:9090`

That proved something was listening on TCP port `9090`.

`'curl -I http://127.0.0.1:9090'` returned `HTTP/1.0 200 OK`, and `'echo $?'` returned `0`.

This taught me one more important idea:

- `'127.0.0.1'` means localhost, which is the same machine only
- it does **not** mean other machines can reach the service
- other machines would need the real server IP, and firewall or security-group rules must allow access

## What port, TCP, socket, and localhost meant in this lab

This lab made the basic terms clearer:

- a **port** is the number that identifies a network service on a machine, here `9090`
- **TCP** is the transport protocol used by this HTTP server
- in simple learning terms, **IP plus port** is the socket address, for example `'127.0.0.1:9090'`
- `'0.0.0.0:9090'` means the service listens on all interfaces on port `9090`
- `'127.0.0.1:9090'` means local access from the same machine

So my corrected understanding was:

- the web server serves files from the web folder
- the port identifies that service on the machine
- localhost tests only local access
- listening on `0.0.0.0` does not automatically mean public access from anywhere

## Deliberate break

I made a backup first:

- `'sudo cp /etc/systemd/system/devops-demo.service /etc/systemd/system/devops-demo.service.bak'`

Then I broke the unit file on purpose by changing the start command to this exact bad line:

- `'ExecStart=/usr/local/bin/devops-demo-start'`

This path did not exist. That was the deliberate break from the schedule.

## Expected symptoms

The expected symptom was:

- the service should fail
- no process should listen on port `9090`

That is exactly what happened.

## Evidence that mattered

The most useful commands were:

- `'systemctl status devops-demo --no-pager'`
- `'journalctl -u devops-demo -n 50 --no-pager'`
- `'ls -l /usr/local/bin/devops-demo*'`
- `'ss -ltnp | grep 9090'`

`'systemctl status devops-demo --no-pager'` showed:

- `Active: failed (Result: exit-code)`
- `ExecStart=/usr/local/bin/devops-demo-start`
- `status=203/EXEC`

The `203/EXEC` part mattered a lot. It showed this was an execution/start-command problem, not a port conflict or an application logic bug.

`'journalctl -u devops-demo -n 50 --no-pager'` showed:

- the old clean run
- the stop of the working service
- repeated failed start attempts
- `Main process exited, code=exited, status=203/EXEC`
- `Start request repeated too quickly`

This taught me that because the unit had `'Restart=on-failure'`, `systemd` kept trying to restart it until it hit the restart limit.

`'ls -l /usr/local/bin/devops-demo*'` showed only:

- `'/usr/local/bin/devops-demo-run'`

There was no `devops-demo-start` file. That directly proved the broken path.

`'ss -ltnp | grep 9090'` returned no matching line, and `'echo $?'` returned `1`. That proved nothing was listening on port `9090` anymore.

## One small thing that could confuse me

After `'sudo systemctl restart devops-demo'`, `'echo $?'` returned `0`, but the service still ended up failed.

So this lab taught me not to trust only the restart command result by itself. I should still check:

- `'systemctl status ...'`
- `'journalctl -u ...'`
- whether the expected port is actually listening

## Root cause

The root cause was the wrong `ExecStart` path in the unit file:

- `'ExecStart=/usr/local/bin/devops-demo-start'`

The real executable script was:

- `'/usr/local/bin/devops-demo-run'`

Because the configured path did not exist, `systemd` could not execute the service start command.

## Fix

The correct fix is:

- change the unit file back to `'ExecStart=/usr/local/bin/devops-demo-run'`
- run `'sudo systemctl daemon-reload'`
- run `'sudo systemctl restart devops-demo'`
- verify with `'systemctl status devops-demo --no-pager'`
- verify with `'ss -ltnp | grep 9090'`
- verify with `'curl -I http://127.0.0.1:9090'`

## Prevention

To prevent this type of failure next time:

- check the exact `ExecStart` path with `'ls -l'` before assuming the app is broken
- remember that `203/EXEC` points to a start-command execution problem
- after a restart, always verify service state and port state, not only the restart command exit code
- keep a backup of the working unit file before editing it

## What I guessed wrong at first

At the start of the clean build, I forgot that `'/opt/devops-demo'` was not writable by my user until I changed ownership again. Later, the biggest lesson was that a service can look like it restarted, but the real truth only appears in `systemctl status`, `journalctl`, and port checks.

## Final Commands I Ran

- `'ssh -i ~/Downloads/tg-key-tokyo.pem ubuntu@43.207.140.32'` -> `'-i'` means use this private key
- `'sudo mkdir -p /opt/devops-demo/web'` -> `'-p'` creates parent folders if needed
- `'printf "<h1>devops demo service is running</h1>\n" > /opt/devops-demo/web/index.html'` -> first failed because of ownership
- `'sudo chown -R ubuntu:ubuntu /opt/devops-demo'` -> `'-R'` means recursive owner change
- `'printf "<h1>devops demo service is running</h1>\n" > /opt/devops-demo/web/index.html'`
- `'sudo tee /usr/local/bin/devops-demo-run > /dev/null <<'EOF' ... EOF'` -> creates the service start script
- `'sudo chmod +x /usr/local/bin/devops-demo-run'` -> `'+x'` adds execute permission
- `'ls -l /usr/local/bin/devops-demo-run'`
- `'file /usr/local/bin/devops-demo-run'`
- `'sudo tee /etc/systemd/system/devops-demo.service > /dev/null <<'EOF' ... EOF'`
- `'sudo systemctl daemon-reload'` -> reread unit files from disk
- `'sudo systemctl start devops-demo'`
- `'sudo systemctl enable devops-demo'`
- `'systemctl status devops-demo --no-pager'` -> `'--no-pager'` prints directly instead of opening a pager
- `'journalctl -u devops-demo -n 20 --no-pager'` -> `'-u'` selects the unit and `'-n 20'` means last 20 lines
- `'ss -ltnp | grep 9090'` -> `'-ltnp'` shows listening TCP ports with numeric output
- `'curl -I http://127.0.0.1:9090'` -> `'-I'` requests headers only
- `'echo $?'` -> `'$?'` shows the previous command exit code
- `'sudo cp /etc/systemd/system/devops-demo.service /etc/systemd/system/devops-demo.service.bak'`
- `'sudo tee /etc/systemd/system/devops-demo.service > /dev/null <<'EOF' ... EOF'` -> overwrote the unit file with the bad `ExecStart` line
- `'sudo systemctl daemon-reload'`
- `'sudo systemctl restart devops-demo'`
- `'echo $?'`
- `'systemctl status devops-demo --no-pager'`
- `'journalctl -u devops-demo -n 50 --no-pager'`
- `'ls -l /usr/local/bin/devops-demo*'`
- `'ss -ltnp | grep 9090'`
- `'echo $?'`
