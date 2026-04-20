# REMIZ — Test Plan
## L4_kolejiste · TwinCAT 3 Simulation

---

## Prerequisites

### 1. TwinCAT setup
1. Open `linka/linka.tsproj` in TwinCAT XAE
2. Right-click **Device 2 (EtherCAT)** → **Simulation Mode**
3. **TwinCAT** menu → **Activate Configuration** → OK
4. **PLC** menu → **Login** → Yes (download)
5. **PLC** menu → **Start**

### 2. Watch window
**PLC** → **Watch** → **Watch 1** — add all variables below.  
To force a value: click the value field → type the value → **F7** (Write values).  
To pulse a BOOL: set `TRUE` → F7 → set `FALSE` → F7.

### 3. Variables to add to Watch window

```
SCADA.systemstav          // PackML state (numeric)
SCADA.techstav            // Technology state — high byte=L4a, low byte=L4b
SCADA.SCADA_START
SCADA.SCADA_STOP
SCADA.SCADA_RESET
SCADA.SCADA_MAN
SCADA.SCADA_ESTOP
SCADA.SCADA_MAN_KLADNY
SCADA.SCADA_MAN_OPACNY
SCADA.SCADA_MAN_ZAVORY
SCADA.SCADA_MAN_IMP_LEV
SCADA.SCADA_MAN_IMP_PRA
SCADA.SCADA_MAN_IMP_ZAD
IO.DI3                    // PRE_KLA  — direction switch forward
IO.DI4                    // PRE_OPA  — direction switch reverse
IO.DI0                    // VYH_LEV  — left switch position feedback
IO.DI1                    // VYH_PRA  — right switch position feedback
IO.DI5                    // TLA_LEV  — manual push button left switch
IO.DI8                    // HRA_PRA  — right gate sensor
IO.DI9                    // HRA_LEV  — left gate sensor (SP1)
IO.DI10                   // HRA_ZAD  — rear gate sensor
IO.DI11                   // HRA_ZAV  — barrier gate sensor (SP2)
IO.DO4                    // ZAVORY   — barriers output (1=down)
IO.DO5                    // KLADNY   — drive forward output
IO.DO6                    // OPACNY   — drive reverse output
IO.DO7                    // IMP_LEV  — left switch impulse output
IO.DO8                    // IMP_PRA  — right switch impulse output
IO.DO9                    // IMP_ZAD  — rear switch impulse output
IO.DO10                   // VYHYBKY  — PLC owns switches
IO.DO11                   // JIZDA    — PLC owns drive
```

### 4. systemstav reference

| Value | State |
|---|---|
| 0 | STOPPED |
| 1 | STARTING |
| 2 | RUNNING |
| 3 | HOLDING |
| 4 | HELD |
| 5 | RESUMING |
| 6 | COMPLETING |
| 7 | COMPLETE |
| 8 | ABORTING |
| 9 | ABORTED |
| 10 | CLEARING |
| 11 | MANUAL |
| 12 | SERVICE |

### 5. techstav reference

| Byte | Bits | Meaning |
|---|---|---|
| High byte | 0–7 | L4a barrier: 0=IDLE, 1=BLOCKED, 2=LIFTING |
| Low byte | 0–7 | L4b switch: 0=IDLE, 1=IMPULSE_C, 2=WAIT_C, 3=IMPULSE_D, 4=WAIT_D |

> In Watch window `techstav` shows as a 16-bit hex value, e.g. `16#0100` means L4a=BLOCKED(1), L4b=IDLE(0).

---

## Test 1 — Power-on / Initial State

**Purpose:** Verify the system starts in a safe, de-energised state.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 1.1 | PLC started, no inputs forced | `systemstav = 0` (STOPPED) | |
| 1.2 | Check all DO outputs | `DO4`–`DO11` all `FALSE` | |
| 1.3 | Check `SCADA.techstav` | `0` (both automata IDLE) | |

---

## Test 2 — AUTO Start / Stop cycle

**Purpose:** Verify STOPPED → STARTING → RUNNING → COMPLETING → COMPLETE → STOPPED.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 2.1 | Pulse `SCADA_START` | `systemstav` goes `1` then `2` (RUNNING) | |
| 2.2 | Check `IO.DO10` (VYHYBKY) | `TRUE` — PLC owns switches | |
| 2.3 | Check `IO.DO11` (JIZDA) | `TRUE` — PLC owns drive | |
| 2.4 | Pulse `SCADA_STOP` | `systemstav` goes `6` (COMPLETING) then `7` (COMPLETE) | |
| 2.5 | Check `IO.DO5`, `IO.DO6` | Both `FALSE` — drive de-energised | |
| 2.6 | Pulse `SCADA_RESET` | `systemstav = 0` (STOPPED) | |

---

## Test 3 — Barrier control (L4a)

**Purpose:** Verify IDLE → BLOCKED → LIFTING → IDLE sequence and 500 ms lift delay.

**Setup:** Start AUTO (Test 2 steps 2.1–2.3). Set `IO.DI3 = TRUE` (forward direction).

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 3.1 | Confirm RUNNING, `IO.DO4 = FALSE` | Barriers up in IDLE | |
| 3.2 | Set `IO.DI11 = TRUE` (loco arrives at barrier) | `techstav` high byte = `1` (BLOCKED), `IO.DO4 = TRUE` | |
| 3.3 | Set `IO.DI11 = FALSE` (loco clears sensor) | High byte = `2` (LIFTING), `IO.DO4` still `TRUE` | |
| 3.4 | Wait 500 ms | High byte = `0` (IDLE), `IO.DO4 = FALSE` | |
| 3.5 | Set `IO.DI11 = TRUE` again during LIFTING (before 500 ms) | High byte jumps back to `1` (BLOCKED) | |
| 3.6 | Set `IO.DI11 = FALSE`, wait 500 ms | Returns to IDLE, `DO4 = FALSE` | |

---

## Test 4 — Switch routing (L4b)

**Purpose:** Verify switches are aligned to correct position based on drive direction, with impulse timing.

**Setup:** RUNNING. Set `IO.DI3 = TRUE` (forward → required position = outer = `1`).

#### 4a — Left switch wrong, right switch wrong

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 4.1 | Set `IO.DI0 = FALSE`, `IO.DI1 = FALSE` (both switches in wrong/inner position) | `techstav` low byte = `1` (IMPULSE_C), `IO.DO7 = TRUE` | |
| 4.2 | Wait 150 ms | Low byte = `2` (WAIT_C), `IO.DO7 = FALSE` | |
| 4.3 | Wait 120 ms (switch travel), `IO.DI1` still `FALSE` | Low byte = `3` (IMPULSE_D), `IO.DO8 = TRUE` | |
| 4.4 | Wait 150 ms | Low byte = `4` (WAIT_D), `IO.DO8 = FALSE` | |
| 4.5 | Wait 120 ms | Low byte = `0` (IDLE) | |

#### 4b — Switches already correct

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 4.6 | Set `IO.DI0 = TRUE`, `IO.DI1 = TRUE` (outer = correct for forward) | Low byte stays `0` (IDLE), no impulse outputs | |

#### 4c — Anti-collision: loco on switch

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 4.7 | Set `IO.DI0 = FALSE` (wrong), `IO.DI9 = TRUE` (loco on left gate) | Low byte stays `0` — no impulse while loco present | |
| 4.8 | Set `IO.DI9 = FALSE` (loco cleared) | Low byte = `1` (IMPULSE_C) starts | |

#### 4d — Reverse direction

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 4.9 | Set `IO.DI3 = FALSE`, `IO.DI4 = TRUE` (reverse direction) | Required position = inner (`0`) |  |
| 4.10 | Set `IO.DI0 = TRUE`, `IO.DI1 = TRUE` (outer = wrong for reverse) | Impulse sequence starts | |

---

## Test 5 — Manual mode

**Purpose:** Verify manual actuator control via SCADA M-variables with anti-collision active.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 5.1 | From STOPPED: pulse `SCADA_MAN` | `systemstav = 11` (MANUAL) | |
| 5.2 | Set `SCADA_MAN_ZAVORY = TRUE` | `IO.DO4 = TRUE` (barriers down) | |
| 5.3 | Set `SCADA_MAN_ZAVORY = FALSE` | `IO.DO4 = FALSE` | |
| 5.4 | Set `SCADA_MAN_KLADNY = TRUE` | `IO.DO5 = TRUE` (forward drive) | |
| 5.5 | Set `SCADA_MAN_OPACNY = TRUE` (with KLADNY still TRUE) | Both `DO5` and `DO6 = FALSE` (mutex) | |
| 5.6 | Set `SCADA_MAN_KLADNY = FALSE`, keep `SCADA_MAN_OPACNY = TRUE` | `IO.DO6 = TRUE` (reverse drive) | |
| 5.7 | Set `SCADA_MAN_IMP_LEV = TRUE` | `IO.DO7 = TRUE` | |
| 5.8 | Set `SCADA_MAN_IMP_PRA = TRUE` | `IO.DO8 = TRUE` | |
| 5.9 | Set `SCADA_MAN_IMP_ZAD = TRUE` | `IO.DO9 = TRUE` | |
| 5.10 | Pulse `SCADA_START` from MANUAL | `systemstav = 0` (STOPPED) | |

---

## Test 6 — E-STOP from RUNNING

**Purpose:** Verify E-STOP immediately de-energises all outputs from any active state.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 6.1 | Start AUTO, confirm RUNNING | `systemstav = 2` | |
| 6.2 | Set `SCADA_ESTOP = TRUE` | `systemstav = 8` (ABORTING) → `9` (ABORTED) | |
| 6.3 | Check ALL DO outputs | `DO4`–`DO11` all `FALSE` | |
| 6.4 | Pulse `SCADA_RESET` while `SCADA_ESTOP` still TRUE | `systemstav` stays `9` — reset blocked | |
| 6.5 | Set `SCADA_ESTOP = FALSE` | No change yet | |
| 6.6 | Pulse `SCADA_RESET` | `systemstav = 10` (CLEARING) → `0` (STOPPED) | |

---

## Test 7 — E-STOP from MANUAL

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 7.1 | Enter MANUAL, activate `SCADA_MAN_KLADNY` | `IO.DO5 = TRUE` | |
| 7.2 | Set `SCADA_ESTOP = TRUE` | `systemstav = 9` (ABORTED), `IO.DO5 = FALSE` | |
| 7.3 | Clear ESTOP, pulse RESET | `systemstav = 0` (STOPPED) | |

---

## Test 8 — Voltage fault (FDI)

**Purpose:** Verify PRE_KLA AND PRE_OPA both TRUE triggers voltage fault and abort.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 8.1 | Start AUTO, confirm RUNNING | `systemstav = 2` | |
| 8.2 | Set `IO.DI3 = TRUE` AND `IO.DI4 = TRUE` simultaneously | `systemstav → 9` (ABORTED), all outputs FALSE | |
| 8.3 | Check `fbDrive.outFault` in Watch | `TRUE` | |
| 8.4 | Check `fbDrive.outFaultCode` | `16#0001` (voltage fault code) | |
| 8.5 | Set `IO.DI3 = FALSE`, `IO.DI4 = FALSE` | Fault stays latched | |
| 8.6 | Pulse `SCADA_RESET` | `systemstav = 0`, fault cleared | |

---

## Test 9 — HOLD / RESUME cycle

**Purpose:** Verify RUNNING → HOLDING → HELD → RESUMING → RUNNING.

| Step | Action | Expected result | Pass? |
|---|---|---|---|
| 9.1 | Start AUTO (RUNNING) | `systemstav = 2` | |
| 9.2 | Pulse `SCADA_MAN` (Hold command from RUNNING) | `systemstav = 3` (HOLDING) → `4` (HELD) | |
| 9.3 | Check drive outputs | `IO.DO5`, `IO.DO6 = FALSE` | |
| 9.4 | Pulse `SCADA_START` (Resume) | `systemstav = 5` (RESUMING) → `2` (RUNNING) | |

---

## Summary checklist

| Test | Description | Pass? |
|---|---|---|
| 1 | Power-on safe state | |
| 2 | AUTO start/stop cycle | |
| 3 | Barrier L4a sequence + lift delay | |
| 4a | Switch routing — both wrong | |
| 4b | Switch routing — already correct | |
| 4c | Switch routing — anti-collision | |
| 4d | Switch routing — reverse direction | |
| 5 | Manual mode + mutex | |
| 6 | E-STOP from RUNNING | |
| 7 | E-STOP from MANUAL | |
| 8 | Voltage fault FDI | |
| 9 | HOLD / RESUME | |

---

*REMIZ Test Plan · L4_kolejiste · TwinCAT 3 Simulation · Build 4026.21*
