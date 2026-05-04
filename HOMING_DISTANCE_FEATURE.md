# Homing Distance Reporting Feature Plan

## Overview
Add minimal, backward-compatible reporting of distance traveled during `$H` homing cycle to enable pen calibration workflow in konin-grbl-app.

## Why This Works
1. Homing stores initial machine position in `sys_position[]` before reset
2. After homing completes, `sys_position[]` is set to new coordinates
3. **Distance = |initial position before home - final position after home|**
4. We calculate this distance and report it in status reports

## Code Changes (20 lines total, ~16 bytes of memory)

### 1. system.h - Add state variable (4 lines)
```c
// In system_t struct, add:
typedef struct {
  // ... existing fields ...
  float homing_distance[N_AXIS];  // Distance traveled during last homing cycle
} system_t;
```

**Why:** Store distance for each axis separately (compact).

### 2. limits.c - Capture distance after homing (5 lines)
Location: End of `limits_go_home()`, right after sys_position is reset (after line ~410)

```c
// After the loop that sets sys_position[idx]:
// Store homing distance for each axis (signed: preserves direction)
float home_position[N_AXIS];
for (idx=0; idx<N_AXIS; idx++) {
  if (cycle_mask & bit(idx)) {
    system_convert_array_steps_to_mpos(home_position, sys_position);
    // Distance with sign: positive = homed in +direction, negative = homed in -direction
    // This represents how far from origin (with direction) the limit switch was
    sys.homing_distance[idx] = home_position[idx];
  }
}
```

**Why:** Keep the sign to preserve direction information (crucial for calibration math). Positive = homed toward +X/+Y, negative = homed toward -X/-Y.

### 3. report.c - Add optional status field (3 lines)
Location: In `report_realtime_status()`, after position reporting (~line 530)

```c
// After reporting machine/work position, add:
if (sys.state == STATE_IDLE) {  // Only report when idle (after homing complete)
  printPgmString(PSTR("|HOMDist:"));
  report_util_axis_values(sys.homing_distance);  // Reuse existing axis formatter
}
```

**Why:** 
- Reports only when idle (not during motion, clean output)
- Uses existing `report_util_axis_values()` function (already handles axis arrays)
- Format is `<...existing...|HOMDist:x.xxx,y.yyy,z.zzz>` with sign preserved
- Unknown fields are safely ignored by standard GRBL clients

## Backward Compatibility

✅ **Standard GRBL tools unaffected:**
- Status report format is extensible (new fields just ignored)
- No changes to command protocol
- No changes to existing fields
- Can be disabled with a compile flag if needed

✅ **Example status report:**
```
<Idle|MPos:10.000,20.000,0.000|FS:0,0|WCO:0.000,0.000,0.000|HOMDist:100.000,100.000,0.000>
```
Standard GRBL tools will parse up to `|WCO:...` and ignore the new `|HOMDist:...` field.

## Memory Cost
- 3 floats × 4 bytes = **12 bytes of RAM**
- Zero cost if not used (cleared by default)

## Code Complexity
- **Total changes: ~20 lines**
- Uses existing utilities (`report_util_axis_values()`, `system_convert_array_steps_to_mpos()`)
- No new algorithms, just data capture

## How konin-grbl-app Will Use This

**Calibration flow:**
1. User clicks "Narysuj krzyż kalibracyjny" → Draw cross at (100, 100)
2. User manually positions pen on cross (stepper position lost)
3. User clicks "Kalibruj offsety XY"
4. Code reads status report: `<...><|HOMDist:150.000,105.000,z>` ← **Signed distance to home!**
5. Calculate offset:
   - Laser drew cross at (100, 100)
   - Limit switch was at (150, 105) from origin (in the direction of homing)
   - User placed pen there manually
   - XY Offset = cross_position - home_distance = (100-150, 100-105) = **(-50, -5)**
   - This means: pen is **50mm left and 5mm back** relative to laser

**Example with negative homing direction:**
- If homing_dir_mask = 0 (homing toward -X, -Y)
- Limit switch at -150, -105 (negative direction)
- HOMDist: -150.000, -105.000
- Offset = (100-(-150), 100-(-105)) = (250, 205) ← correct calculation

## Verification
After implementation, test with:
```
# Home machine
$H

# Check status report includes new field
?
# Should show: <Idle|....|HOMDist:...>
```

## Branch Strategy & Deployment

### Default Behavior (Safety First)
- **DEFAULT:** Master branch stays UNCHANGED (backward compatible)
- **FEATURE:** New feature on branch `report-homing-distance`
- **RATIONALE:** Extensive testing needed before production. Clients using standard GRBL won't break.

### Installation Script Updates
File: `scripts/start-with-wrapper.sh` and build scripts should support:
```bash
# Option 1: Standard GRBL (default, no homing distance reporting)
./grbl-setup/compile-grbl.sh master

# Option 2: With homing distance reporting (for konin-grbl-app calibration)
./grbl-setup/compile-grbl.sh report-homing-distance
```

Add to `config.h`:
```c
// Enable homing distance reporting in status reports (default: disabled)
// #define ENABLE_HOMING_DISTANCE_REPORT
```

When `report-homing-distance` branch is used, this flag is defined.
When `master` branch is used, feature is compiled out (zero overhead).

## Alternative: Compile-Time Optional
If worried about breaking anything:
```c
#ifdef ENABLE_HOMING_DISTANCE_REPORT
  // Add distance field to status report
#endif
```
Can be set in config.h, defaults to disabled if desired.
