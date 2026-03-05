# Bridge Report: SYIL LNC6800 → GSK 218MC Probing Port

**Target:** GSK CNC218MC-V on CanCam Z540, wireless probe (6.74mm ball, R=3.37mm)
**Date:** 2026-02-22
**Sources:** `/oem_macros` (24 factory programs), SYIL macro library (root), `/post/fanuc_gsk_218mc_probing_V7.cps`, 218M01/218M02 manuals, machine-confirmed reference (12 proven O9100-series programs)

---

## 1. OEM Probing System Summary

### 1.1 Probe Arm / Disarm Sequence

**Workpiece probing** (O83211, O83213, O83214, O83240):
```
M48;      (enable measurement mode — arms controller skip logic)
M46;      (turn probe on — arms wireless probe receiver)
G04 X1;   (1-second settle — MANDATORY, wireless signal stabilization)
...
M49;      (off measurement mode — disarms both)
```

**Tool-setter measurement** (O81021, O81022, O81023):
```
M48;      (enable measurement mode)
M47;      (valid for cutter jump signal input — tool-setter-specific)
M37;      (start air blast at tool setter)
G04 X1;   (1-second settle)
...
M38;      (close air blast)
M49;      (off measurement mode)
```

> **Machine-confirmed:** M47 does NOT appear in any workpiece probing macro. Its exact function on this controller is unconfirmed. **Do not use M47 for workpiece probing.** Only M48 + M46 + G04 X1 are needed.

### 1.2 G31 Skip Probing

OEM syntax: `G31 X[target] F[feedrate];` — identical to G01, modal state (G90/G91) applies.

- Non-modal: active for one block only.
- G40 (cancel cutter comp) **must** be active before any G31 or Alarm 0036 fires.
- If probe does NOT trigger, axis travels to commanded endpoint and stops — **no alarm generated**.
- `#3004=2` enables skip input; `#3004=0` disables. The OEM macros set/clear this around each slow-pass probe.

**Two-pass pattern** (from O83026):
1. Fast pass: `G31 X[endpoint] F#119;`
2. Settle: `G04 P100;` (100 ms)
3. Retract: `G01 X[startback] F[fast];`
4. Slow pass: `#3004=2; G31 X[endpoint] F#27; G04 P100; #3004=0;`
5. Read: use `#5016` / `#5017` / `#5018`

### 1.3 Skip Position Variables

Machine-confirmed (218M01.pdf p.136):

| Variable | Description | Coordinate System |
|----------|-------------|-------------------|
| `#5016` | X at trigger | Workpiece (current WCS) |
| `#5017` | Y at trigger | Workpiece (current WCS) |
| `#5018` | Z at trigger | Workpiece (current WCS) |
| `#5019` | A at trigger | Workpiece (current WCS) |

> **Critical:** `#5016–#5018` are workpiece coordinates, NOT machine coordinates.
> **`#5061–#5063` do not exist on this control.** Do not use them.

### 1.4 Current Position Variables

| Variable | Description | Coord System |
|----------|-------------|-------------|
| `#5011` | X | Workpiece (ABSOT) |
| `#5012` | Y | Workpiece (ABSOT) |
| `#5013` | Z | Workpiece (ABSOT) |
| `#5006` | X | Machine (ABSMT) |
| `#5007` | Y | Machine (ABSMT) |
| `#5008` | Z | Machine (ABSMT) |

### 1.5 WCS Offset Variables (readable and writable)

Machine-confirmed (218M01.pdf p.137-138). Stride = 5 variables per WCS:

| WCS | X      | Y      | Z      | A      |
|-----|--------|--------|--------|--------|
| G54 | #5206  | #5207  | #5208  | #5209  |
| G55 | #5211  | #5212  | #5213  | #5214  |
| G56 | #5216  | #5217  | #5218  | #5219  |
| G57 | #5221  | #5222  | #5223  | #5224  |
| G58 | #5226  | #5227  | #5228  | #5229  |
| G59 | #5231  | #5232  | #5233  | #5234  |

**Dynamic WCS selection formula (confirmed):**
```
#130 = #19 - 54;               (index: 0=G54, 1=G55 ... 5=G59)
#131 = 5206 + [#130 * 5];     (base variable for X offset)
```
Indirect assignment confirmed working: `#[#131 + 1] = #[#131 + 1] + offset;`

**Extended WCS (S=1–50):** `#7001+(S-1)*5` = X, `#7002+(S-1)*5` = Y, `#7003+(S-1)*5` = Z.
O9100-series **do not support extended WCS** — only G54–G59.

> **Warning:** O83032 (factory) uses `#[5201+[(S-53)*5]]` as its base, which gives a different variable set.
> O83213 (factory) has a confirmed WCS stride bug. **Do not use it as a reference.**

### 1.6 Tool Offset Variables

| Range | Description |
|-------|-------------|
| `#[1500+T]` | Tool T length offset (base) |
| `#[1800+T]` | Tool T length wear |
| `#[2100+D]` | Tool D radius offset (base) |
| `#[2400+D]` | Tool D radius wear |

### 1.7 Probe Calibration Variables (persistent #500–#999)

Set by O83103/O83104 (probe calibration routines):

| Variable | Description |
|----------|-------------|
| `#500`   | X-axis probe radius (calibrated) |
| `#501`   | Y-axis probe radius |
| `#502`   | X eccentricity (probe center vs spindle center) |
| `#503`   | Y eccentricity |
| `#119`   | Fast probe speed (3000 mm/min metric) |
| `#123`   | Position threshold (0.05mm metric) |
| `#129`   | Unit multiplier (1.0 metric) |

### 1.8 No-Trigger Error Detection

OEM pattern from O83026: compare `#5016` to the commanded endpoint.
If G31 does not trigger, `#5016` ends up **at the commanded endpoint**, not near the start.

```
#120 = [target];
G31 X#120 F[feed];
IF[ABS[#5016 - #120] LT 0.5] GOTO900;   (0.5mm threshold = no trigger)
```

---

## 2. SYIL LNC6800 Macro Library Summary

### 2.1 Language and Runtime

The SYIL macros use LNC6800 (LinuxCNC-derivative) proprietary syntax — **not** standard Fanuc Macro B. The following LNC-specific primitives have **no direct equivalent** on GSK 218MC:

| LNC Primitive | GSK Equivalent |
|---------------|----------------|
| `R_SKIP[0,1]` (1=triggered) | Compare `#5016` to endpoint |
| `R_SKIP[0,201]` (X at skip) | `#5016` |
| `R_SKIP[0,202]` (Y at skip) | `#5017` |
| `R_SKIP[0,203]` (Z at skip) | `#5018` |
| `R_MACH_COOR[0,1/2/3]` | `#5006` / `#5007` / `#5008` |
| `R_SYS_INFO[0,2]` (current tool) | `#4027` (system variable) |
| `R_TOOL_DATA[0,T,3]` (diameter) | `#[2100+T]*2` |
| `R_TOOL_DATA[0,T,203]` (length) | `#[1500+T]` |
| `W_G53G59_COOR[0,WCS,axis,val]` | Write `#[5206+...]` directly |
| `W_G54EXP_COOR[0,ext,axis,val]` | Write `#[7001+...]` directly |
| `W_TOOL_DATA[0,T,3,val]` | Write `#[2100+T]` directly |
| `W_TOOL_DATA[0,T,203,val]` | Write `#[1500+T]` directly |
| `FIX_CUT_OR_ON / OFF` | `#3004=2 / #3004=0` |
| `ALARM["msg"]` | `#3000=1(msg)` — parens NOT brackets |
| `MENU_ADD[...]` / `MENU[...]` | Not supported — omit or adapt |
| `MSG_OK[...]` | `M00` (optional stop with operator cue) |
| `G31 G91 P2 X[d] F[f]` | `G91; G31 X[d] F[f]; G90;` |
| `@100–@116` (persistent globals) | `#500–#599` (common variables) |
| `@996–@999` (result globals) | `#600–#699` (common variables) |
| `@5109` (calibration store) | `#800` (common variable, chosen) |

### 2.2 Probe Arm / Disarm

SYIL only uses:
- `M19` — orient spindle before probing
- `M20` — unlock spindle after probing
- No M46/M47/M48/M49 — probe activation is implicit on LNC

**GSK equivalent:** `M48; M46; G04 X1.0;` to arm, `M49;` to disarm.
GSK does not use M19/M20 for wireless probe operations.

### 2.3 WCS Setting Logic

SYIL argument convention:
- `A#1` = WCS number (e.g., 54 for G54, 54.5 for G54P5 extended)
- `#114 = ROUND[[#1 - FIX[#1]] * 10]` extracts the extended offset index

GSK equivalent for G54–G59 only:
```
#130 = #19 - 54;
#131 = 5206 + [#130 * 5];
#[#131] = new_X_offset;
```

### 2.4 Safe Z

SYIL uses `G28 G91 Z0` (machine home) or `G53 G90 G00 Z0` for retracts.
No explicit "safe Z" variable — retract height is controller home.

GSK equivalent: same `G91 G28 Z0;` works identically on GSK 218MC.

### 2.5 Two-Pass Architecture

SYIL pattern:
1. `G31 G91 P2 X[dist] F[fast]`
2. Check `R_SKIP[0,1] == 1`
3. `G91 G01 X-[backoff]`
4. `FIX_CUT_OR_ON`
5. `G31 G91 P2 X[dist] F[slow]`
6. `FIX_CUT_OR_OFF`
7. Read `R_SKIP[0,201]`

GSK equivalent (confirmed pattern):
1. `G31 X[endpoint] F[fast];`
2. Check `ABS[#5016 - endpoint] > 0.5` for trigger
3. `G00 X[retract];`
4. `#3004=2;`
5. `G31 X[endpoint2] F[slow];`
6. `G04 P100;`
7. `#3004=0;`
8. Read `#5016`

---

## 3. Fusion 360 Post Processor Analysis

**File:** `post/fanuc_gsk_218mc_probing_V7.cps`

### 3.1 What It Outputs

**Probe section header** (emitted at start of each probe operation):
```
M48;        (enable measurement mode)
M46;        (arm probe)
G04 X1;     (1-second settle)
G49;        (cancel tool length comp)
G43 H0;     (zero tool length — redundant with G49, confirmed unnecessary)
```

**Probe section footer:**
```
M49;        (disarm probe)
```

**Each probe cycle** calls an O9100-series program via G65:
- `G65 P9100 S[wcs] I[dir] R[mm];` — single X surface
- `G65 P9101 S[wcs] I[dir] R[mm];` — single Y surface
- `G65 P9102 S[wcs] R[mm];` — single Z surface
- `G65 P9110 S[wcs] I[dir] J[dir] R[mm];` — inner corner
- `G65 P9111 S[wcs] I[dir] J[dir] R[mm];` — outer corner
- `G65 P9120 S[wcs] D[mm] E[mm] Z[mm] T[mm] R[mm];` — rect boss
- `G65 P9121 S[wcs] D[mm] E[mm];` — rect pocket
- `G65 P9122 S[wcs] D[mm] Z[mm] T[mm] R[mm];` — circ boss
- `G65 P9123 S[wcs] D[mm];` — circ bore

### 3.2 WCS Mapping

Fusion WCS 1–6 → G54–G59 only. No extended WCS support.

```
fusionWCS = currentSection.probeWorkOffset   (target to UPDATE)
workOffset = fusionWCS + 53                  (passed as S parameter)
```

The post uses `probeWorkOffset` (the WCS to update), not `workOffset` (the driving WCS for positioning). This distinction is critical — confirmed correct in V7.

### 3.3 Direction Convention

**Surface probing (O9100/O9101) — INVERTED:**
- Fusion "positive" approach (probe moves +X) → post outputs `I = -1.0`
- Fusion "negative" approach (probe moves -X) → post outputs `I = +1.0`

**Corner probing (O9110/O9111) — NOT inverted:**
- Fusion "positive" approach → post outputs `I = +1.0`
- Fusion "negative" approach → post outputs `I = -1.0`

### 3.4 Units

All probe parameter values are converted to mm before output. GSK 218MC operates in metric only (G21). The post forces G21.

### 3.5 Assumptions Made by the Post

1. O9100–O9123 programs exist on the machine.
2. Machine operates in metric (G21) exclusively.
3. Only G54–G59 WCS (indices 1–6) are valid probe targets.
4. Probe radius R is always passed in mm.
5. `protectedProbeMove()` positions tool to `(x, y, z - depth)` before macro call — not to retract height.
6. Z retract after probing is handled within individual O91xx programs (G91 G28 Z0 in O9102).

### 3.6 Known Gaps

- **T1 M06 conflict:** When probe is already loaded, a spurious tool-change call faults. Currently removed manually.
- **G43 H0 is redundant:** G49 alone sufficient. Not harmful but wastes a block.
- **No M19/M20:** Not needed for wireless probe — confirmed correct omission.

---

## 4. Key Differences: SYIL vs OEM vs Post

| Aspect | SYIL (LNC6800) | OEM GSK Factory | Post Output (V7) |
|--------|---------------|----------------|-----------------|
| **Language** | LNC proprietary (C-style `@vars`, `R_SKIP[]`, `W_G53G59_COOR[]`) | Fanuc Macro B compatible | JavaScript (CPS) outputs Macro B |
| **Probe arm** | M19 (orient only) | M48 + M46 + G04 X1 | M48 + M46 + G04 X1 |
| **Probe disarm** | M20 (orient unlock) | M49 | M49 |
| **M47** | Not used | Tool-setter only | Never output |
| **G31 syntax** | `G31 G91 P2 X[d] F[f]` (LNC P2 flag) | `G31 X[target] F[f]` (standard) | Inside O91xx macros |
| **Skip trigger check** | `R_SKIP[0,1] == 1` | `ABS[#5016 - endpoint] > threshold` | Inside O91xx macros |
| **Skip position read** | `R_SKIP[0,201/202/203]` | `#5016 / #5017 / #5018` | Inside O91xx macros |
| **WCS write** | `W_G53G59_COOR[0,wcs,axis,val]` | `#[5206+(wcs-54)*5+ax] = val` | O91xx uses direct write |
| **Extended WCS** | G54.5 → `W_G54EXP_COOR[0,5,...]` | `#[7001+(S-1)*5]` | Not supported in V7 |
| **WCS variable for G54 X** | Abstracted | `#5206` (stride 5 from #5206) | `#5206` in O91xx |
| **Tool length read** | `R_TOOL_DATA[0,T,203]` | `#[1500+T]` | Not needed in O91xx |
| **Two-pass control** | `FIX_CUT_OR_ON/OFF` | `#3004=2 / #3004=0` | Inside O91xx |
| **Error alarm** | `ALARM["text"]` | `#3000=1(text)` | Inside O91xx |
| **Inline comments** | `// comment` | **None allowed** | N/A |
| **Line endings** | LNC doesn't care | **CRLF required** | Generated by post |
| **Global variables** | `@100–@999` | `#500–#999` (common) | Hardcoded in O91xx |
| **G65 A/B params** | A=#1, B=#2 | **A and B NOT recognized** — use I/J/D/E | Post uses I/J/D/E ✓ |
| **Local var persistence** | Cleared by LNC | **#1–#26 persist between M30 runs** | Handled by sentinel |
| **WCS stride** | Abstracted | 5 variables per WCS, base #5206 | `#131 = 5206+[(#19-54)*5]` |

---

## 5. Proposed Adapter/Wrapper Layer Approach

### 5.1 Architecture Overview

The target architecture is already in production. The O9100-series programs ARE the adapter layer — they translate the Fanuc-style G65 calling convention (from Fusion/post) into confirmed GSK 218MC Macro B patterns.

```
┌─────────────────────────────────────────────────────┐
│  Fusion 360 CAM                                     │
│  (sets up probe operations, WCS targets)            │
└────────────────────────┬────────────────────────────┘
                         │ .nc file
┌────────────────────────▼────────────────────────────┐
│  fanuc_gsk_218mc_probing_V7.cps (post processor)    │
│  - Emits M48/M46/G04/M49 headers                    │
│  - Maps Fusion WCS 1-6 → S54-S59                    │
│  - Converts all values to mm                        │
│  - Outputs G65 P91xx calls                          │
└────────────────────────┬────────────────────────────┘
                         │ G65 P91xx calls
┌────────────────────────▼────────────────────────────┐
│  O9100-series (adapter layer) — on controller       │
│  - Standard G65 parameter interface                 │
│  - Two-pass probing (G31 fast+slow)                 │
│  - Dynamic WCS selection (#131 formula)             │
│  - Direct WCS variable writes (#5206+...)           │
│  - Dual-mode: M30 standalone / M99 subroutine       │
└─────────────────────────────────────────────────────┘
```

### 5.2 Key Design Rules for O9100-Series

These rules are confirmed by machine testing (12 proven programs):

1. **CRLF line endings** (`\r\n`) — LF-only causes Alarm on line 1.
2. **Semicolon after every line**, including `O9100;`.
3. **No inline comments.** Comments on their own line only: `(THIS IS A COMMENT);`
4. **Max 2–3 header comment lines.** Large comment blocks cause controller bog-down (no alarm, just slow).
5. **No A or B parameters in G65.** Use I (#4), J (#5) for direction. Use D (#7), E (#8) for dimensions.
6. **Alarm syntax:** `#3000 = 1(message);` — parentheses, never square brackets.
7. **Sentinel pattern** for local variable persistence:
   ```
   (at top) IF[#3 EQ 1] THEN #19 = #0;
   (at top) IF[#3 EQ 1] THEN #3 = #0;
   (at M30) #3 = 1;
   ```
8. **Store machine coords before WCS write; return via G53:**
   ```
   #111 = #5006;  #113 = #5007;  #114 = #5008;
   ... (WCS write) ...
   G53 G00 Z#114;  G53 G00 X#111;  G53 G00 Y#113;
   ```
9. **Dual-mode ending:**
   ```
   IF[#19 NE #0] THEN #140 = 1;   (subroutine mode)
   IF[#19 EQ #0] THEN #140 = 0;   (standalone mode)
   ...
   IF[#140 EQ 1] GOTO9999;
   #3 = 1;
   M30;
   N9999;
   M99;
   ```
10. **G40 must be active** before any G31 block.

### 5.3 Standard Parameter Interface

Unified across all O9100-series programs:

| Param | Variable | Meaning | Default |
|-------|----------|---------|---------|
| S | #19 | Target WCS (54–59) | 54 (G54) |
| I | #4  | X direction (1.0=from+X, -1.0=from-X) | — |
| J | #5  | Y direction (1.0=from+Y, -1.0=from-Y) | — |
| D | #7  | X width or diameter (mm) | — |
| E | #8  | Y width (mm) | — |
| R | #18 | Probe ball radius (mm) | 3.37 |
| F | #9  | Slow feed (mm/min) | 100 |
| W | #23 | Fast feed (mm/min) | 500 |
| Q | #17 | Z travel or clearance distance (mm) | 10 |
| Z | #26 | Z drop distance (mm, negative) | -10 |
| T | #20 | Clearance beyond nominal (mm) | 10 |

### 5.4 WCS Update Formula (Confirmed)

```
#130 = #19 - 54;               (0=G54, 1=G55, ..., 5=G59)
#131 = 5206 + [#130 * 5];     (X variable for target WCS)

(Update X — approach from +X side, I=1.0):
#[#131] = #[#131] + #5016 - #18;

(Update X — approach from -X side, I=-1.0):
#[#131] = #[#131] + #5016 + #18;

(Unified — any approach direction, I = ±1.0):
#[#131] = #[#131] + #5016 - [#4 * #18];

(Update Y):
#[#131 + 1] = #[#131 + 1] + #5017 - [#5 * #18];

(Update Z — always from above):
#[#131 + 2] = #[#131 + 2] + #5018 - #18;
```

---

## 6. First 3 Safest Macros to Port

These are already implemented and machine-proven as O9100, O9101, O9102. This section documents them as templates for future work and for test validation.

---

### 6.1 MACRO 1: PROBEZ → O9102 (Single Z Surface)

**Why safest:** Single axis, single touch, no XY motion during probing, probe approaches from above. Zero risk of unexpected lateral collision.

**SYIL original:** `PROBEZ` — Argument A (WCS), Argument B (probe distance, negative).

**GSK O9102 interface:** `G65 P9102 S[wcs] R[radius_mm];`
Post positions above surface at correct XY before calling.

**Logic skeleton:**
```
O9102;
(Single Z surface probe - approach from above)
IF[#3 EQ 1] THEN #19=#0; IF[#3 EQ 1] THEN #3=#0;
IF[#19 NE #0] THEN #140=1; IF[#19 EQ #0] THEN #140=0;
IF[#19 EQ #0] THEN #19=54;
IF[#18 EQ #0] THEN #18=3.37;
IF[#23 EQ #0] THEN #23=500;
IF[#9  EQ #0] THEN #9=100;
IF[#17 EQ #0] THEN #17=10;
#111=#5006; #113=#5007; #114=#5008;
G40 G90;
G#19;
#130=#19-54; #131=5206+[#130*5];
#120=#5013-#17;         (endpoint = current Z - travel)
G31 Z#120 F#23;         (fast pass)
IF[ABS[#5018-#120] LT 0.5] GOTO900;
G00 Z[#5018+2.0];       (retract 2mm)
#3004=2;
G31 Z#120 F#9;          (slow pass)
G04 P100;
#3004=0;
IF[ABS[#5018-#120] LT 0.5] GOTO900;
#[#131+2]=#[#131+2]+#5018-#18;   (update Z WCS)
G53 G00 Z#114;
IF[#140 EQ 1] GOTO9999;
#3=1; M30;
N900; #3000=1(Z PROBE NO TRIGGER);
N9999; M99;
```

**Test Plan:**

| Step | Action | Pass Criterion |
|------|--------|----------------|
| 1 | Load tool as T99 probe, zero fixture in G54 by hand | G54 Z manually set |
| 2 | Run standalone: `G65 P9102` (no params) | No alarm; DRO G54.Z changes by ~surface height |
| 3 | Run three times consecutively | Z result repeatable within 0.01mm |
| 4 | Measure resulting G54 Z offset against indicator | ≤ 0.02mm error |
| 5 | Run via G65 with explicit WCS: `G65 P9102 S55` | G55.Z updates; G54 unchanged |
| 6 | Position probe 5mm short of surface; run — should alarm | Alarm 1 fires, no crash |
| 7 | Feed override 25%, run Fusion-posted probing Z operation | Correct WCS Z set |

---

### 6.2 MACRO 2: PROBEX → O9100 (Single X Surface)

**Why second:** Single axis, but requires direction logic (I parameter) and lateral approach. Slightly more complex than Z but no Z cycling.

**SYIL original:** `PROBEX` — Argument A (WCS), Argument B (signed distance).

**GSK O9100 interface:** `G65 P9100 S[wcs] I[dir] R[radius_mm];`

**Direction convention (post-confirmed):**
- Fusion "positive" approach (probe moves +X) → post sends `I=-1.0`
- Meaning: I=-1.0 = probe approaching from the +X side

**Logic skeleton:**
```
O9100;
(Single X surface probe)
IF[#3 EQ 1] THEN #19=#0; IF[#3 EQ 1] THEN #4=#0; IF[#3 EQ 1] THEN #3=#0;
IF[#19 NE #0] THEN #140=1; IF[#19 EQ #0] THEN #140=0;
IF[#19 EQ #0] THEN #19=54;
IF[#18 EQ #0] THEN #18=3.37;
IF[#23 EQ #0] THEN #23=500;
IF[#9  EQ #0] THEN #9=100;
IF[#17 EQ #0] THEN #17=10;
IF[#4  EQ #0] THEN #4=-1;      (default: approach from +X)
#111=#5006; #113=#5007; #114=#5008;
G40 G90;
G#19;
#130=#19-54; #131=5206+[#130*5];
#120=#5011-[#4*#17];            (endpoint = current X - I*travel)
G31 X#120 F#23;                 (fast pass)
IF[ABS[#5016-#120] LT 0.5] GOTO900;
G00 X[#5016+[#4*2.0]];         (retract 2mm away from surface)
#3004=2;
#121=#5011-[#4*[#17+1.0]];
G31 X#121 F#9;                  (slow pass)
G04 P100;
#3004=0;
IF[ABS[#5016-#121] LT 0.5] GOTO900;
#[#131]=#[#131]+#5016-[#4*#18];  (update X WCS)
G53 G00 Z#114;
G53 G00 X#111; G53 G00 Y#113;
IF[#140 EQ 1] GOTO9999;
#3=1; M30;
N900; #3000=1(X PROBE NO TRIGGER);
N9999; M99;
```

**Test Plan:**

| Step | Action | Pass Criterion |
|------|--------|----------------|
| 1 | Set a known X surface (machined block, measured with indicator) | Reference X known to ±0.005mm |
| 2 | Run standalone, approach from +X (`I=-1`): `G65 P9100` | No alarm; G54.X updates |
| 3 | Run three times from same position | X result repeatable within 0.01mm |
| 4 | Compare G54.X to indicator reading | ≤ 0.02mm |
| 5 | Flip direction: `G65 P9100 I1` (approach from -X) | Same surface, same result |
| 6 | Run Fusion "probing-x positive" operation | Post sends `I=-1`, correct G54.X set |
| 7 | Run Fusion "probing-x negative" operation | Post sends `I=1`, correct G54.X set |
| 8 | No-trigger test: position 3mm short of travel | Alarm 1 fires, no crash |

---

### 6.3 MACRO 3: PROBEY → O9101 (Single Y Surface)

**Why third:** Mirrors O9100 exactly but for Y axis. Validates that the J parameter and #5017 path works identically.

**GSK O9101 interface:** `G65 P9101 S[wcs] I[dir] R[radius_mm];`

> Note: Post sends direction in I for both X and Y single-surface probes. O9101 internally maps I (#4) to the Y direction.

**Logic skeleton:** Identical to O9100 with X→Y substitutions:
- `#5016` → `#5017`
- `G31 X#120` → `G31 Y#120`
- `#[#131]` → `#[#131+1]`
- `#5011` → `#5012`
- `G53 G00 X#111` → `G53 G00 Y#113`

**Test Plan:**

| Step | Action | Pass Criterion |
|------|--------|----------------|
| 1 | Set known Y surface | Reference Y known ±0.005mm |
| 2 | Standalone `G65 P9101` (approach from +Y, I=-1 default) | G54.Y updates, no alarm |
| 3 | Three consecutive runs | Y repeatable within 0.01mm |
| 4 | Compare to indicator | ≤ 0.02mm |
| 5 | Flip: `G65 P9101 I1` (from -Y) | Same surface, same result |
| 6 | Run Fusion "probing-y" cycle | Correct WCS.Y updated |
| 7 | No-trigger test | Alarm 1 fires |

---

## 7. GSK 218MC Confirmed Facts (From Machine Testing)

The following are confirmed by 12 proven O9100-series programs on this exact machine:

### G-Code Syntax
- Semicolon required after every line including O-number.
- **CRLF (`\r\n`) required.** LF-only → Alarm on line 1. The #1 gotcha for programmatic generation.
- Comments: own line, parentheses only: `(COMMENT);` — no inline comments after G-code.
- Consecutive comment blocks (15+ lines) → controller bog, no alarm. Keep headers to 2-3 lines.
- G40 must be active before G31 or Alarm 0036.

### G65 Parameters
- A and B parameters are **NOT recognized** by GSK 218MC. Positive values silently dropped; negative values leak through a quirk. Use I/J for direction.
- Recognized addresses: `G X Y Z R I J K F H M S T P Q D E U V W C`

### Variables
- Local variables `#1–#26` **persist between M30 program ends**. Defaults break after first run. Use sentinel pattern.
- `#5016–#5018` = skip positions in workpiece coords. Confirmed. `#5061–#5063` = do not exist.
- WCS stride = 5. G54 base = `#5206`. Formula: `#[5206 + (wcs - 54) * 5 + axis]`.

### Probe
- Probe ball: 6.74mm diameter, radius = **3.37mm** (ball bearing, measured 2026-02-20).
- M48 + M46 + G04 X1 = arm. M49 = disarm. No M47 needed.
- G31 no-trigger: axis reaches endpoint, no alarm. Detect by comparing `#5016` to endpoint.

### Rotary (A-axis)
- Positive A: top rotates away from operator.
- **Never use `G00 A[absolute]` with probe near workpiece.** Always incremental (`G91 G00 A[delta] G90`) after safe Z retract.

---

## Appendix: Macro Cross-Reference

| SYIL Macro | OEM Analog | O9100-Series | Status |
|------------|------------|--------------|--------|
| PROBEZ | O83240, O83211 | O9102 | Proven ✓ |
| PROBEX | O83211 | O9100 | Proven ✓ |
| PROBEY | O83211 | O9101 | Proven ✓ |
| PROBEXSLOT / PROBEYSLOT | O83213 | O9121 (pocket) | Proven ✓ |
| PROBEXWEB / PROBEYWEB | O83213 | O9120 (boss) | Proven ✓ |
| PROBEBORE | O83214 | O9123 | Proven ✓ |
| PROBECIRCULARBOSS | O83214 | O9122 | Proven ✓ |
| PROBEINSIDECORNER | O83211 | O9110 | Proven ✓ |
| PROBEOUTSIDECORNER | O83211 | O9111 | Proven ✓ |
| PROBECONFIG | O81012 / O81019 | Inlined in each O91xx | Complete |
| PROTECTEDMOVE | O83210 | Not ported (not needed by post) | — |
| CALIBRATEPROBEZ | O83101 | Not ported | — |
| CALIBRATEPROBEBLOCK | O83102 | Not ported | — |
| CALIBRATETOOLSET | O81021 / O81023 | Not ported | — |
