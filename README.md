# sandbox-run

**Run code you don't trust — a stranger's repo, its test suite, something an AI agent wrote — walled off from the rest of your Linux machine. No Docker, no VM: one bash script and the systemd you already have.**

```bash
curl -fsSLO https://raw.githubusercontent.com/KhaiB10/sandbox-run/v1.0.0/sandbox-run
less sandbox-run                               # read it first — it's one short script that runs as root
sudo install -m 755 sandbox-run /usr/local/bin/
sudo sandbox-run setup                         # once: creates the unprivileged `sandbox` user
W=$(sudo sandbox-run workdir myproject)        # a work dir; put the code here
sudo git clone https://github.com/someone/thing "$W/thing" && sudo chown -R sandbox: "$W"
sudo sandbox-run --dir myproject --net -- sh -c 'cd thing && npm ci'   # installs need the network
sudo sandbox-run --dir myproject -- sh -c 'cd thing && npm test'       # tests don't
```

## What the code inside can and can't do

| | inside the sandbox |
|---|---|
| user | the unprivileged `sandbox` user; `sudo` and setuid are blocked |
| files | its own work dir only. `/home`, `/root`, `/var/lib`, `/mnt`, `/media`, `/etc/shadow`, `/etc/ssh` and other sandboxes are hidden; the rest of the OS is read-only |
| network | **localhost only** by default (a local Ollama or database is reachable, the internet is not). `--net` for full access, `--no-net` for none |
| resources | memory 8G (no swap), 4 CPU cores, 1024 processes, 1 hour — all adjustable |
| kernel | no module loading, no kernel tunables, no namespaces, no clock or hostname changes |

## Prove it on your machine

Don't trust this README — run the escape tests. Each one runs *inside* the sandbox and tries to get out:

```
$ sudo sandbox-run check
  ✓ runs as the unprivileged sandbox user        ok
  ✓ can write its own work directory             ok
  ✓ cannot list your home (/home/you)            blocked
  ✓ cannot read /root                            blocked
  ✓ cannot read /etc/shadow                      blocked
  ✓ cannot see /var/lib                          blocked
  ✓ cannot see sibling work dirs                 blocked
  ✓ cannot write the OS (/etc, /usr)             blocked
  ✓ cannot use sudo                              blocked
  ✓ cannot reach the internet (default)          blocked
  ✓ memory limit holds (asks for 2x --mem)       blocked
  ✓ time limit holds                             blocked
  ✓ cannot see other sandboxes' work dirs        blocked
  controls — the same probes with the wall removed must succeed, or the tests above prove nothing:
  ✓ control: internet reachable with --net       ok
  ✓ control: 128M fits under --mem 512M          ok
  ✓ control: sleep finishes under --time 30s     ok

16 passed, 0 failed
```

The controls matter: a "blocked" test would also pass if the probe itself were broken. Each control runs the same probe with the wall lifted and must succeed.

## Options

```
sandbox-run [--dir NAME] [--net | --no-net] [--hide PATH]... [--mem 8G] [--cpus 4] [--time 1h] [--env K=V]... -- <command>
```

Environment: `SANDBOX_BASE` (default `/srv/sandbox-run`), `SANDBOX_USER` (default `sandbox`).

## What this is not

This is systemd's service sandboxing (the same walls system services run behind), not a virtual machine. The code shares your kernel, so a kernel exploit could escape it. It is a strong fence against code that is careless, nosy or buggy — a test suite that deletes `~`, a dependency that reads your SSH keys, an agent that runs the wrong command — and a much weaker one against malware written to break out. For the second, use a VM.

Needs: Linux with systemd 247+, run with `sudo`. Built and tested on Ubuntu 24.04 (systemd 255).

## Where it came from

It was built to run strangers' repositories safely while reproducing bugs for open-source projects, where it ran pnpm installs, a 55-test suite and an AI coding agent without any of them seeing the host.

## License

MIT
