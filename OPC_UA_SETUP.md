# OPC UA Setup Guide — RPA SoftPLC → SCADA

## Overview

The PLC project uses TwinCAT 3. SCADA variables are declared in
`linka/L4_kolejiste/GVLs/SCADA.TcGVL` with fixed `%M` memory addresses.
The `{attribute 'OPC.UA.DA' := '1'}` pragma has already been added to all
SCADA variables in this repo — no PLC logic changes are needed.

OPC UA is provided by the **TwinCAT 3 Function Package TF6100**.
The server runs on the same machine as the SoftPLC (TwinCAT/BSD or Windows).

---

## Step 1 — Check / Activate TF6100 License

1. Open **TwinCAT XAE** (Visual Studio with TwinCAT shell).
2. In the Solution Explorer open **SYSTEM → License**.
3. Look for `TF6100 OPC-UA Server` in the list.
   - If listed and active → proceed to Step 2.
   - If missing → you need to purchase or activate a 7-day trial:
     - Right-click the license → **Activate 7 Day Trial License**
     - For production: generate a **License Request File** and send it to Beckhoff.
4. After license activation, **restart TwinCAT** (TwinCAT → Restart TwinCAT).

---

## Step 2 — Enable the OPC UA Server in TwinCAT

1. In TwinCAT XAE, go to **SYSTEM → Real-Time → Settings** and confirm the
   runtime is in **Config Mode** (blue icon in system tray).
2. Open **SYSTEM → TwinCAT OPC UA** node (appears after TF6100 install).
   - If the node is missing, install TF6100 on the SoftPLC machine first
     (download from Beckhoff website, run installer).
3. In the OPC UA configuration:
   - **Port**: leave at `4840` (standard OPC UA port).
   - **Endpoint**: `opc.tcp://<softplc-ip>:4840`
   - **Security**: for a local/test network, `None / None` is fine.
     For production use `Basic256Sha256` + certificates.
   - **Anonymous login**: enable for initial testing; add username/password later.

---

## Step 3 — Rebuild and Activate the PLC Configuration

The `{attribute 'OPC.UA.DA' := '1'}` pragmas are already in the code.
After rebuild, TwinCAT auto-generates the symbol export file.

1. In TwinCAT XAE open the solution for **L4_kolejiste**
   (or the combined solution if you use one).
2. **Build → Build Solution** (F7).
   - Check Output window for 0 errors.
3. **TwinCAT → Activate Configuration** → confirm with **OK**.
4. Switch runtime to **Run Mode** (green icon in system tray).
5. Confirm PLC task is running: **PLC → L4_kolejiste → Online**.

---

## Step 4 — Verify OPC UA Symbol Export

TwinCAT generates a symbol file at (on the SoftPLC machine):

```
C:\TwinCAT\3.1\Boot\Plc\Port_854\TcOpcUaServer.xml   (or similar path)
```

Open this file and search for `SCADA_START` — you should see an entry like:

```xml
<Symbol name="PLC1.SCADA.SCADA_START" ... />
```

If the symbols are missing, check:
- The pragma `{attribute 'OPC.UA.DA' := '1'}` is directly above the variable (no blank line between pragma and variable).
- The GVL has `{attribute 'qualified_only'}` at the top — this is already set.
- Rebuild was done after adding pragmas.

---

## Step 5 — Test with a Free OPC UA Client

Before connecting SCADA, verify with **UaExpert** (free, from Unified Automation):

1. Download and install UaExpert on any PC on the same network.
2. Add server: `opc.tcp://<softplc-ip>:4840`
3. Connect → browse the address space:
   ```
   Root → Objects → PLC1 → SCADA → SCADA_START
                                  → SCADA_RESET
                                  → techstav
                                  → systemstav
                                  → ...
   ```
4. Drag variables to the **Data Access View** and verify live values.
5. Test a write: double-click `SCADA_START`, set value to `TRUE`, confirm
   the PLC reacts (state should move from STOPPED → STARTING).

---

## Step 6 — Connect SCADA

Configure your SCADA tool's OPC UA client with these node IDs:

### Control inputs (SCADA writes → PLC)

| Variable | OPC UA Node ID | PLC Address |
|---|---|---|
| START | `ns=2;s=PLC1.SCADA.SCADA_START` | `%MX100.0` |
| RESET | `ns=2;s=PLC1.SCADA.SCADA_RESET` | `%MX100.1` |
| STOP | `ns=2;s=PLC1.SCADA.SCADA_STOP` | `%MX100.2` |
| MAN mode | `ns=2;s=PLC1.SCADA.SCADA_MAN` | `%MX100.3` |
| E-STOP | `ns=2;s=PLC1.SCADA.SCADA_ESTOP` | `%MX100.4` |
| Man: drive+ | `ns=2;s=PLC1.SCADA.SCADA_MAN_KLADNY` | `%MX101.0` |
| Man: drive- | `ns=2;s=PLC1.SCADA.SCADA_MAN_OPACNY` | `%MX101.1` |
| Man: barriers | `ns=2;s=PLC1.SCADA.SCADA_MAN_ZAVORY` | `%MX101.2` |
| Man: switch L | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_LEV` | `%MX101.3` |
| Man: switch R | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_PRA` | `%MX101.4` |
| Man: switch rear | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_ZAD` | `%MX101.5` |

### Status outputs (PLC writes → SCADA reads)

| Variable | OPC UA Node ID | PLC Address | Meaning |
|---|---|---|---|
| Tech state | `ns=2;s=PLC1.SCADA.techstav` | `%MW104` | High byte = barrier state (0/1/2), Low byte = switch state (0–4) |
| System state | `ns=2;s=PLC1.SCADA.systemstav` | `%MW106` | PackML state (see table below) |

### systemstav values (PackML)

| Value | State | Description |
|---|---|---|
| 0 | STOPPED | Idle, waiting for START |
| 1 | STARTING | Initialization in progress |
| 2 | RUNNING | Automatic cycle active |
| 3 | HOLDING | Hold requested |
| 4 | HELD | Held, waiting for resume |
| 5 | RESUMING | Returning to RUNNING |
| 6 | COMPLETING | Graceful stop in progress |
| 7 | COMPLETE | Cycle finished |
| 8 | ABORTING | Fault abort in progress |
| 9 | ABORTED | Fault latched, waiting for RESET |
| 10 | CLEARING | Reset in progress |
| 11 | MANUAL | Manual mode active |
| 12 | SERVICE | Service mode (PIN required) |

### techstav byte breakdown

```
techstav (WORD = 16 bits)
  High byte (%MB105) = L4a barrier state
    0 = IDLE      (no train at barrier)
    1 = BLOCKED   (train detected, barrier lowered)
    2 = LIFTING   (barrier lifting after train passed)

  Low byte (%MB104) = L4b switch router state
    0 = IDLE
    1 = IMPULSE_C  (sending switch impulse, direction C)
    2 = WAIT_C     (waiting for switch confirmation)
    3 = IMPULSE_D  (sending switch impulse, direction D)
    4 = WAIT_D     (waiting for switch confirmation)
```

---

## Step 7 — Firewall

On the SoftPLC machine (Windows or TwinCAT/BSD), open port 4840 TCP inbound:

**Windows (PowerShell as Administrator):**
```powershell
New-NetFirewallRule -DisplayName "TwinCAT OPC UA" -Direction Inbound `
  -Protocol TCP -LocalPort 4840 -Action Allow
```

**TwinCAT/BSD:** configure via the BSD firewall (`pf`) or the TwinCAT network
configuration panel.

---

## Troubleshooting

| Problem | Check |
|---|---|
| Can't connect in UaExpert | Ping the SoftPLC IP. Check port 4840 is open (firewall). Check TwinCAT is in Run Mode. |
| Variables not visible in browser | Rebuild solution after adding pragmas. Check `TcOpcUaServer.xml` was regenerated. |
| Node ID `ns=2;s=PLC1.SCADA...` not found | The PLC instance name may differ from `PLC1` — check in TwinCAT XAE under the PLC project name. |
| Write from SCADA has no effect | Confirm `outEnable_Man` or `outEnable_Auto` is TRUE — most commands are gated by machine state. |
| `systemstav` stays at 0 after START | Check HW E-STOP is not active, and `SCADA_ESTOP` (%MX100.4) is FALSE. |

---

## Notes on Other Line Modules (L1, L5, L6)

Only **L4_kolejiste** has a SCADA GVL at this time. L1, L5, L6 use only
`TAGS.TcGVL` for raw I/O. If you need OPC UA on those modules, create a
`SCADA.TcGVL` for each (same pattern as L4) and add the OPC UA pragmas.

Their PLC ports:
- L1_otocny_stul → Port 851
- L5_prisavka    → Port 852
- L6_plosina     → Port 853
- L4_kolejiste   → Port 854
