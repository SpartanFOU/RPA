# OPC UA Setup — TwinCAT 3 → mySCADA
## REMIZ / L4_kolejiste

---

## Overview

The PLC project runs on a Beckhoff TwinCAT 3 controller. SCADA variables are declared in
`linka/L4_kolejiste/GVLs/SCADA.TcGVL` with fixed `%M` memory addresses and the
`{attribute 'OPC.UA.DA' := '1'}` pragma already applied to all exported variables.

OPC UA connectivity is provided by **TwinCAT 3 Function Package TF6100 OPC-UA Server**,
configured via the standalone **TwinCAT OPC UA Configurator** application.
The SCADA frontend is **mySCADA**.

---

## Step 1 — Activate TF6100 License

1. Open **TwinCAT XAE** (Visual Studio with TwinCAT shell).
2. In Solution Explorer: **SYSTEM → License**.
3. Find `TF6100 OPC-UA Server` in the list and activate:
   - **Trial:** right-click → **Activate 7 Day Trial License**
   - **Full:** generate a License Request File and submit to Beckhoff
4. Restart TwinCAT after activation.

---

## Step 2 — Install and Open the TwinCAT OPC UA Configurator

The **TwinCAT OPC UA Configurator** is a standalone application installed as part of the
TF6100 package. It is separate from TwinCAT XAE.

1. After TF6100 installation, open **TwinCAT OPC UA Configurator** from the Start menu.
2. The main view shows a list of configured OPC UA server instances.

---

## Step 3 — Create a Server Instance

1. In the Configurator, click **Add Server** (or the `+` button).
2. Give the server a name (e.g. `REMIZ_Server`).
3. Leave the port at **4840** (standard OPC UA port).
4. Confirm — the new server instance appears in the list.

---

## Step 4 — Configure Security (Non-Encrypted)

For lab use, anonymous non-encrypted access is sufficient.

1. Open the server instance → go to the **Security** or **Endpoints** tab.
2. Set **Security Policy** to `None`.
3. Set **Authentication** to `Anonymous` (no username/password required).
4. Save the configuration.

> For production deployments use `Basic256Sha256` with certificates and username/password.

---

## Step 5 — Point the Server to the TwinCAT Project Symbols (.tcm file)

The OPC UA server needs to know which PLC symbols to expose. This is done by referencing
the compiled TwinCAT Module Configuration file (`.tcm`).

1. In the server configuration, find the **Symbol/Namespace** or **Data Source** section.
2. Browse to the `.tcm` file of the compiled PLC project. Typical path:

   ```
   C:\TwinCAT\3.1\Boot\Plc\Port_854\
   ```

   Select the `.tcm` file for **L4_kolejiste** (Port 854).

3. The Configurator will load the symbol list from the file.
4. In the **Namespace** view, verify that the SCADA variables are visible:

   ```
   PLC1 → SCADA → SCADA_START
                → SCADA_RESET
                → SCADA_STOP
                → SCADA_MAN
                → SCADA_ESTOP
                → SCADA_MAN_KLADNY
                → SCADA_MAN_OPACNY
                → SCADA_MAN_ZAVORY
                → SCADA_MAN_IMP_LEV
                → SCADA_MAN_IMP_PRA
                → SCADA_MAN_IMP_ZAD
                → techstav
                → systemstav
   ```

5. Confirm / apply — the namespace is now managed.

---

## Step 6 — Allow Port 4840 in Windows Firewall

On the machine running TwinCAT / the OPC UA server, open port 4840 TCP inbound.

**PowerShell (run as Administrator):**
```powershell
New-NetFirewallRule -DisplayName "TwinCAT OPC UA" -Direction Inbound `
  -Protocol TCP -LocalPort 4840 -Action Allow
```

Or manually via **Windows Defender Firewall → Advanced Settings → Inbound Rules → New Rule →
Port → TCP → 4840 → Allow**.

---

## Step 7 — Connect mySCADA

1. Open **mySCADA** and go to **Project Settings → Remote Servers** (or equivalent).
2. Add a new **OPC UA** remote server with:
   - **Endpoint URL:** `opc.tcp://<plc-ip-address>:4840`
   - **Security:** None / Anonymous
3. Connect — mySCADA should list the available nodes.
4. Map variables to mySCADA tags:

### Control inputs (mySCADA writes → PLC)

| mySCADA Tag | OPC UA Node ID | PLC Address | Description |
|---|---|---|---|
| START | `ns=2;s=PLC1.SCADA.SCADA_START` | `%MX100.0` | Start command |
| RESET | `ns=2;s=PLC1.SCADA.SCADA_RESET` | `%MX100.1` | Reset / clear fault |
| STOP | `ns=2;s=PLC1.SCADA.SCADA_STOP` | `%MX100.2` | Stop command |
| MAN | `ns=2;s=PLC1.SCADA.SCADA_MAN` | `%MX100.3` | Manual mode select |
| E-STOP | `ns=2;s=PLC1.SCADA.SCADA_ESTOP` | `%MX100.4` | Software E-STOP |
| MAN_KLADNY | `ns=2;s=PLC1.SCADA.SCADA_MAN_KLADNY` | `%MX101.0` | Manual: drive forward |
| MAN_OPACNY | `ns=2;s=PLC1.SCADA.SCADA_MAN_OPACNY` | `%MX101.1` | Manual: drive reverse |
| MAN_ZAVORY | `ns=2;s=PLC1.SCADA.SCADA_MAN_ZAVORY` | `%MX101.2` | Manual: barriers down |
| MAN_IMP_LEV | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_LEV` | `%MX101.3` | Manual: left switch |
| MAN_IMP_PRA | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_PRA` | `%MX101.4` | Manual: right switch |
| MAN_IMP_ZAD | `ns=2;s=PLC1.SCADA.SCADA_MAN_IMP_ZAD` | `%MX101.5` | Manual: rear switch |

### Status outputs (PLC → mySCADA reads)

| mySCADA Tag | OPC UA Node ID | PLC Address | Description |
|---|---|---|---|
| techstav | `ns=2;s=PLC1.SCADA.techstav` | `%MW104` | Technology state (see below) |
| systemstav | `ns=2;s=PLC1.SCADA.systemstav` | `%MW106` | PackML system state (see below) |

---

## State Reference Tables

### systemstav (%MW106) — PackML states

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

### techstav (%MW104) — Technology sequence states

```
techstav (WORD = 16 bits)
  High byte (%MB105) = L4a barrier state
    0 = IDLE      (no train at barrier crossing)
    1 = BLOCKED   (train detected, barrier lowered)
    2 = LIFTING   (barrier lifting after train passed, 500 ms delay)

  Low byte (%MB104) = L4b switch router state
    0 = IDLE
    1 = IMPULSE_C  (left switch impulse active)
    2 = CHECK_C    (verifying left switch position after impulse)
    3 = IMPULSE_D  (right switch impulse active)
    4 = CHECK_D    (verifying right switch position after impulse)
```

Example: `techstav = 16#0100` → L4a = BLOCKED (1), L4b = IDLE (0)

---

## Troubleshooting

| Problem | Check |
|---|---|
| mySCADA cannot connect | Ping the PLC IP. Confirm port 4840 is open in firewall. Check TwinCAT is in Run Mode. |
| Variables not visible in namespace | Confirm the correct `.tcm` file is referenced in the Configurator. Rebuild and re-activate the PLC project. |
| Node ID not found | The PLC instance name may differ from `PLC1` — verify in TwinCAT XAE under the PLC project name. |
| Write from mySCADA has no effect | Confirm the machine is in the correct state — most commands are gated by `outEnable_Man` or `outEnable_Auto` in `FB_MachineControl`. |
| `systemstav` stays at 0 after START | Check E-STOP is not active: HW NC input must be closed (TRUE), and `SCADA_ESTOP` (%MX100.4) must be FALSE. |

---

*REMIZ Project — OPC UA Setup · TF6100 Standalone Configurator · mySCADA integration*
