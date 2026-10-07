# Plan: open the FBPA_PLL window in cmpunlocker (for 170tune compatibility)

## Context

The goal is to be able to run **170tune** (https://github.com/cachenetics/170tune) on a mixed
fleet (GPU0/1 = 8 GB `0x20C2`, GPU3 = 10 GB `0x2082`) unlocked via this `cmpunlocker` repo.

### Key finding (premise correction)

170tune does **not** want the `asm64-cmpunlocker` `--mclk-ndiv` driver-baked clock. It *actively
refuses* it:

> "A driver built with cmpunlocker's `--mclk-ndiv` flag bakes a non-stock clock into its own init...
> `status`, `preflight`, `snapshot-stock` and every mclk/persist command that touches the memory
> clock check for this and **refuse**." (170tune `README.md`, `tools/170tune:599,685-691`)

170tune sets the memory clock (NDIV) itself, **live from userspace over BAR0** (`tools/hbm_mclk.c`),
so the card boots stock and a bad value is a reboot away. What it actually needs is the **FBPA_PLL
register window opened to host writes** plus a genuinely stock clock. Per-GPU NDIV is then handled
entirely by 170tune.

### The gap is a privilege mask, not the overclock

| PLM opened at boot | `cmpunlocker` (this repo) | `asm64-cmpunlocker` | needed by 170tune's `hbm_mclk` |
|---|---|---|---|
| FBPA_MEM `0x009a0148` | yes | yes | - |
| FBPA_PLL0/1/2 `0x009a3c7c/80/84` | **no** | yes (`cmpunlock.c:624`, unconditional) | **yes - this is the blocker** |
| baked non-stock NDIV | (n/a) | only if `--mclk-ndiv` passed | **no - 170tune refuses** |

`hbm_mclk` reads the unicast FBPA0 PLL `0x00903c7c` and checks the host-write bit `0x10`
(`hbm_mclk.c:44,69,167`). On stock `cmpunlocker` that window is fenced, so the COEFF reads back as a
PRI error (`0xBADF...`) and `hbm_mclk set` exits 2 ("unlock not applied"). Opening the broadcast
FBPA_PLL PLMs `0x009a3c7c/80/84 -> 0xffffffff` at the existing post-booter point unblocks the
unicast window, exactly as the `asm64` lineage does.

### Decision

**Path B** - patch this `cmpunlocker` repo to open the FBPA_PLL window, keeping `cmpunlocker` as the
unlocker. (Path A would be to just install `asm64-cmpunlocker` without `--mclk-ndiv`; not chosen.)

## Goal

`cmpunlocker`'s post-booter PLM table also opens FBPA_PLL0/1/2 (`0x009a3c7c/80/84 -> 0xffffffff`),
so 170tune's live `hbm_mclk` lever can read/write the unicast PLL COEFF over BAR0. Card boots
**stock** (no baked clock), so 170tune does not refuse.

## Non-goals (explicitly NOT doing)

- No port of `asm64`'s `--mclk-ndiv` / `--mclk-timings`.
- No post-GSP hook.
- No per-GPU baked clock / registry-per-device logic.

170tune refuses a baked clock and does NDIV live and per-card itself (plus per-card `snapshot-stock`,
live timings via `fbpa_regs`, and always-boot-stock is its own escape hatch).

## Changes

### File 1: `driver/patches/sec2-postbl-plm-ss-cfg.patch`

All edits are inside the hunk at **patch line 69**, header `@@ -4821,6 +4857,143 @@` (b-side/new
count = **143**).

**Edit 1a - add 3 table rows** (patch lines 91-92). From:

```
+            { 0x0000c848U, 0xffffffffU, "PJTAG_SEC_PLM" },
+        };
```

To:

```
+            { 0x0000c848U, 0xffffffffU, "PJTAG_SEC_PLM" },
+            { 0x009a3c7cU, 0xffffffffU, "FBPA_PLL0" },
+            { 0x009a3c80U, 0xffffffffU, "FBPA_PLL1" },
+            { 0x009a3c84U, 0xffffffffU, "FBPA_PLL2" },
+        };
```

(+3 added lines; the only count-affecting edit.)

**Edit 1b - de-magic the loop bound** (patch line 100). From:

```
+        for (plmIdx = 0; plmIdx < 12; plmIdx++)
```

To:

```
+        for (plmIdx = 0; plmIdx < NV_ARRAY_ELEMENTS(plmTable); plmIdx++)
```

Single `+` line changed in place -> **count-neutral**; future PLM rows never need a bound edit.
(`NV_ARRAY_ELEMENTS` is the same macro the `asm64` lineage uses on this tree, so it is defined.)

Variant if preferred: keep the literal and just change `12` -> `15` instead (still count-neutral).

**Edit 1c - fix unified-diff counts/offsets.** Only the b-side grows (a-side untouched). The edited
hunk's new-count `143 -> 146`; every later hunk's `+` start shifts **+3**:

| Hunk (patch line) | header now            | header after           |
|-------------------|-----------------------|------------------------|
| 69                | `-4821,6 +4857,143`   | `-4821,6 +4857,146`    |
| 213               | `-5662,6 +5816,84`    | `+5816 -> +5819`       |
| 298               | `-5672,6 +5904,9`     | `+5904 -> +5907`       |
| 308               | `-5682,7 +5917,7`     | `+5917 -> +5920`       |
| 317               | `-5694,8 +5929,48`    | `+5929 -> +5932`       |
| 368               | `-5709,6 +5984,69`    | `+5984 -> +5987`       |
| 438               | `-6612,6 +6950,51`    | `+6950 -> +6953`       |

(`patch -p1` is often tolerant of a miscount, but the counts are set correctly so the patch stays
valid for both `patch` and `git apply`.)

### File 2: `common/constants.yaml`

Under `unlocks.privilege_masks.registers`, right after `fbpa_plm` (line 54), add:

```yaml
      fbpa_pll0_plm:  { addr: "0x009a3c7c", value: "0xffffffff" }
      fbpa_pll1_plm:  { addr: "0x009a3c80", value: "0xffffffff" }
      fbpa_pll2_plm:  { addr: "0x009a3c84", value: "0xffffffff" }
```

`tools/read-constants.py` `present()` hex-normalizes, so `0x009a3c7c`/`0x009a3c7cU` in the patch
satisfies these; keeps the constants <-> patch cross-check green. No new patch file, so it does not
trip the "built-but-not-declared" check (`read-constants.py:63-68`).

### File 3: docs (keep dependents consistent)

- `docs/INSTALLATION.md` and root `README.md`: note the unlock now opens the FBPA_PLL window ->
  compatible with 170tune's live memory lever (stock boot, needs `iomem=relaxed`).
- `docs/HISTORY.md`: one changelog line.

## Verification gate (before declaring done)

1. Fresh-extract `patch -p1 --dry-run` the edited patch against the cached `610.43.02` tarball in
   `driver/.build/` -> applies clean (catches any diff-count error).
2. `read-constants.py` for each profile (`8gb` / `10gb` / `mixed`) -> passes.
3. Full `driver/build.sh` on `610.43.02` **and `615.71.09`** -> confirms hunk context still matches
   the newer tree.
4. On-card (user): install patched driver stock (no clock flag) -> `sudo 170tune preflight` reports
   the PLL window open; `hbm_mclk get` returns a real NDIV, not `0xBADF`.

## Risk / rollback

- **Risk: low.** Opening PLMs is additive and mirrors the `asm64` lineage's unconditional table. The
  only real hazard is a unified-diff count error, which step 1's dry-run catches before install.
- **Rollback:** revert the 2 source files via git. The card always boots stock, so there is no
  recovery risk from the driver change itself.

## User-side prerequisites (outside this repo)

- `sudo ./install.sh` from 170tune (builds its helpers).
- `iomem=relaxed` on the kernel command line (GRUB) so userspace can map BAR0.
- NVML + a CUDA toolkit for the sm_80 probes / integrity gate.
