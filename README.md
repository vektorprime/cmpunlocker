# cmpunlocker — fbpa-pll-170tune fork

> **Why we created this branch:** we wanted the current cmpunlocker changes (e.g. 74 SM) while also
> having the overclocking capability of
> [asm64-cmpunlocker](https://github.com/asm64-hooligan/cmpunlocker) (asm64-hooligan's fork).

Upstream cmpunlocker never opens the FBPA_PLL privilege windows, so no live HBM memory-clock lever
works on it; the asm64-cmpunlocker lineage opens them (and bakes a clock in with `--mclk-ndiv`), but
has not kept up with upstream's unlock work (full SM config, geometry, BAR1, persistence). This
branch merges the two: it stays on upstream cmpunlocker and adds the FBPA PLL/MEM window opens from
the asm64 lineage, while deliberately **not** baking any clock into the driver — the card boots at
stock, and the memory clock is set live from userspace (e.g. via
[170tune](https://github.com/vektorprime/170tune)), so a bad value is a reboot away instead of a
reinstall. See [170tune compatibility (fork)](#170tune-compatibility-fork) below.

## 170tune compatibility (fork)

This fork additionally opens the FBPA privilege windows that live memory control needs (the same
windows the asm64-cmpunlocker lineage opens) — something the upstream repo does not provide.
No clock is baked into the driver; the card boots stock, which is exactly what 170tune requires.
Pair it with [vektorprime/170tune](https://github.com/vektorprime/170tune) (`fbpa-first-active`):

- **8 GB (`0x20C2`)**: full windows — FBPA_MEM, broadcast FBPA_PLL0/1/2, and unicast
  FBPA0 PLL. The live NDIV lever, timings, and refresh all work (`preflight` green).
- **10 GB (`0x2082`)**: FBPA_MEM + broadcast FBPA_PLL0/1/2. The unicast FBPA0 row is
  skipped on this SKU — its window is orphaned in hardware (the SEC2 booter itself
  faults touching it, killing GSP init if forced). SM, timings, and refresh levers
  work; the 170tune fork drives the memory lever through the first live FBPA instead.

---

## Proof of Concept

Below are memory and performance results after applying the unlock:

<table>
  <tr>
    <td><b>Memory Unlock Results</b></td>
  </tr>
  <tr>
    <td><img alt="memory unlock" src="https://github.com/user-attachments/assets/ae062bd8-e3a7-4e73-b9a4-fbcde53f3c7b" width="100%" style="max-width: 900px;" /></td>
  </tr>
</table>

<table>
  <tr>
    <td><b>Performance Benchmarks (<a href="https://github.com/ProjectPhysX/OpenCL-Benchmark">OpenCL-Benchmark</a>)</b></td>
  </tr>
  <tr>
    <td><img alt="performance benchmarks" src="https://github.com/user-attachments/assets/2501506d-420f-4014-9574-b1bd0290eb60" width="100%" style="max-width: 900px;" /></td>
  </tr>
</table>

---

## Requirements

- Linux (x86-64)
- Root access
- NVIDIA CMP 170HX
- **nvidia-open 610.xx.xx+ already installed** (libs + firmware)
- Kernel headers matching the running kernel (`linux-headers-$(uname -r)` / `kernel-devel`)
- Secure Boot disabled (patched modules are unsigned)
- Network access on first install (downloads matching stock `open-gpu-kernel-modules` sources)
- Python 3 (used at build time to select 8GB/10GB geometry)

---

## Install

To install cmpunlocker, run the following command:

```bash
sudo ./install.sh
```

To force a certain memory profile, use the `--profile` option:

```bash
sudo ./install.sh --profile=8gb    # 8GB card → 64GB unlock
sudo ./install.sh --profile=10gb   # 10GB card → 40GB unlock
```

Then perform a reboot.

## What Gets Unlocked

<table>
  <tr>
    <th>Feature</th>
    <th>Status</th>
  </tr>
  <tr>
    <td>Full SM compute throughput (SS0/SS1)</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>Memory geometry (64GB on 8GB cards, 40GB on 10GB cards)</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>PCIe Gen 2 speeds</td>
    <td>Working ✓</td>
  </tr>
    <tr>
    <td>Error-Correcting Code (ECC) DRAM and SRAM </td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>Full BAR1 Size (64GB)</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>JTAG (Host2Jtag register access)</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>VFIO-based passthrough</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>GPU profiling</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>Persistence across reboot (patched modules)</td>
    <td>Working ✓</td>
  </tr>
  <tr>
    <td>Live HBM memory-clock lever (FBPA PLL/MEM windows open, fork)</td>
    <td>Working ✓</td>
  </tr>
</table>

---

## Uninstall

To uninstall cmpunlocker, run the following command:

```bash
sudo ./remove.sh --yes
```

Then perform a reboot.

## Contributions

Please read [docs/CONTRIBUTING.md](https://github.com/amoghmunikote/cmpunlocker/blob/master/docs/CONTRIBUTING.md) before opening a PR.

## Support & Community

Having issues? Need help? Join our [Discord community](https://discord.gg/CdHSakKSFv) to discuss with other users and get support.
