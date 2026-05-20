# sparktop

A live, `top`-style GPU monitor for the NVIDIA DGX Spark (GB10).

Pure Python stdlib — no dependencies. It reads telemetry from `nvidia-smi`
(NVML), which is the only interface that exposes GPU utilization, power, and
clocks on the GB10 superchip (there are no Tegra-style sysfs load nodes).

## Usage

```
sparktop            # live dashboard, refresh every 1s  (q or Ctrl-C to quit)
sparktop -n 2       # refresh every 2 seconds
sparktop --once     # print one snapshot and exit (scriptable, no ANSI when piped)
sparktop --no-color # disable color
```

It's installed at `~/.local/bin/sparktop` (a symlink to this script).

## What it shows

- Per-GPU compute and memory utilization bars (color-coded by load)
- Temperature, power draw/limit, SM and memory clocks, fan
- GPU compute processes, sorted by memory use

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
