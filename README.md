# sparktop

A live, `top`-style GPU monitor for the NVIDIA DGX Spark (GB10).

Pure Python stdlib — no dependencies. It reads telemetry from `nvidia-smi`
(NVML), which is the only interface that exposes GPU utilization, power, and
clocks on the GB10 superchip (there are no Tegra-style sysfs load nodes).

## Install

```
brew install brianmatzelle/tap/sparktop
```

Or run the script directly — it's pure stdlib, so any Python 3.11+ works:

```
git clone https://github.com/brianmatzelle/sparktop && ./sparktop/sparktop
```

## Usage

```
sparktop            # live dashboard, refresh every 1s  (q or Ctrl-C to quit)
sparktop -n 2       # refresh every 2 seconds
sparktop --once     # print one snapshot and exit (scriptable, no ANSI when piped)
sparktop --no-color # disable color

sparktop --ports 3000-3009,8080,25565   # which ports to watch
sparktop --no-ports        # hide the ports pane
sparktop --no-services     # hide the services pane
```

## What it shows

- Per-GPU compute and memory utilization bars (color-coded by load)
- Temperature, power draw/limit, SM clock, and P-state
- **Services** — the systemd units you wrote, system and user, with state and PID
- **Ports** — what's listening on the common dev ports, and who owns it
- GPU compute processes, sorted by memory use

Panes are listed in that order and shrink to fit: on a short pane each one is
truncated with a `… +N more` marker, and the lowest-priority panes drop out
entirely before the GPU bars ever scroll off the top.

## Services

"Yours" means the unit file lives outside the distro's directories
(`/usr/lib/systemd`, `/lib/systemd`, `/run/systemd`) — i.e. in
`/etc/systemd/system` or `~/.config/systemd/user`. Snap-generated units are
skipped by name. Failed and activating units sort to the top.

Note that third-party packages sometimes install into `/etc/systemd/system`
too, so a few vendor units can show up; widen `VENDOR_UNIT_DIRS` in the script
if any of them bother you.

## Ports

TCP and UDP listeners are matched against the watched ranges, which default to
the common dev ports:

```
3000-3009,4000-4009,4321-4322,5000-5009,5173-5183,5432,6379,7860,
8000-8199,8888,9090,11434,19132,25565,25575,27017
```

Override them with `--ports` or `$SPARKTOP_PORTS`. UDP rows are tagged
`/udp`; TCP is unmarked. `BIND` is `*` when the port is bound to all
interfaces (highlighted, since that's reachable off-box), `local` when it's
loopback-only, or the specific address otherwise.

Owners are resolved in four steps, because no single source covers everything:

1. `ss` names the process for sockets you own.
2. That PID's `/proc/<pid>/cgroup` gives either the owning systemd unit or the
   docker container it belongs to — the file is world-readable, so this works
   for other users' processes too. A `network_mode: host` container lands here.
3. Published container ports are proxied by root-owned `docker-proxy`
   processes, whose cgroup only ever says `docker.service`, so those are
   matched by port number against `docker ps`. Both cases are tagged `·docker`.
4. Failing all that, `/proc/net/{tcp,udp}` still exposes the socket's uid, so
   the row shows `?` plus the owning user, e.g. `? ·root`.

Step 4 is where a root-owned socket lands: the kernel only lets you map a
socket back to a process through `/proc/<pid>/fd`, which needs ownership of
that process. So `? ·root` means "a root process holds this port" — run
sparktop as root to see which one, or run the service as your own user.

`ss`, `systemctl` and `docker` are all optional — a pane just disappears when
its source isn't there.

## If you see "Driver/library version mismatch"

That means the loaded NVIDIA kernel module and the userspace library are on
different versions (common right after a driver update). `nvidia-smi` itself
won't work until they match — **reboot** to fix it:

```
sudo reboot
```

`sparktop` detects this case and tells you the same thing instead of crashing.

## Testing without a working GPU

Point it at a stub that mimics `nvidia-smi`'s CSV output:

```
SPARKTOP_NVIDIA_SMI=/path/to/fake-nvidia-smi sparktop --once
```
