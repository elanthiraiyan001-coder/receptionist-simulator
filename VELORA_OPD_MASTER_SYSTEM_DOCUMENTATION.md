# 🏥 Velora OPD Queue Intelligence Engine — Master System Documentation & Implementation Blueprint (v7.0)

> **Document Purpose**: This single unified master document contains **both** the complete technical reference notes explaining every detail of the system AND the exact self-contained prompt blueprint required to recreate this entire system in a new AI chat.

---

## 📑 PART 1: MASTER RE-CREATION PROMPT FOR NEW CHATS

*If you want an AI agent in a new chat to build this exact system from scratch, copy and paste the prompt below:*

```text
Please build the complete Velora OPD Queue Intelligence Engine v7.0 in two HTML files: receptionist.html and doctor.html.

Requirements:
1. Technology Stack: HTML, Vanilla CSS (dark glassmorphism theme with #07090e background, Plus Jakarta Sans & Outfit Google Fonts), and Vanilla JavaScript with LocalStorage cross-tab telemetry under the key 'VELORA_OPD_SHARED_STATE_V7' polling at 100ms intervals.
2. Initial State: Pre-load 20 App-Booked Patients (P-1 to P-20) starting at simulation time 08:40:00 AM (31200 seconds). Baseline ACT is 10.0m.
3. Queue Priority Hierarchy: EMERGENCY (E-1, #1 position), VIP (V-1, gold priority + doctor alert banner), DIAGNOSTIC_RETURN (#2 position), RESTROOM_RETURN, APP (P-1..P-20), WALKIN (W-1..).
   * VIP Reference (`V-1`) Gate Approval Workflow:
     - When receptionist registers a VIP patient (`type === 'VIP'`), the patient enters `status = 'PENDING_VIP_APPROVAL'` and shows tag `⏳ PENDING DOCTOR APPROVAL`. The patient is **100% excluded from the Doctor Cabin Lobby until doctor decides**.
     - An alert is broadcast to `doctor.html` (`pendingVIPAlert = { token, name }`).
     - Glowing gold notification banner pops up on doctor console.
     - **`⚡ APPROVE VIP PRIORITY`**: Doctor approves VIP priority. Dynamic lookup sets `p.prio = 'VIP_APPROVED'` (Rank 2) and `p.status = 'VERIFIED_IN_LOBBY'`. The VIP patient NOW enters the Doctor Cabin Lobby as Position #1 priority (right behind emergencies)!
     - **`⏱️ DEFER TO REGULAR QUEUE`**: Doctor defers VIP status. Dynamic lookup sets `p.prio = 'REGULAR'` (Rank 5) and `p.status = 'VERIFIED_IN_LOBBY'`, and **moves the patient to the VERY END of the regular queue** so they are 100% NOT called next!
4. Rolling ACT Engine: Baseline 10m, clamped between 4m-30m. Implement Exponential Moving Average (0.7 * old + 0.3 * actual). Fast visits <= 0.5*ACT, slow visits >= 1.5*ACT. Use the 3-Consecutive Patient Sample Rule (3 fast/slow visits in a row required before shifting ACT). Exempt Emergency/VIP (divide duration by 1.5), Lab returns, Restroom returns, Breaks, and Delays from the ACT average.
5. Static Reporting Time & 30m Lead Buffer: Calculate Lead Buffer = min(2.0 * ACT, 30 mins). Reporting Time = Estimated Turn - Lead Buffer. Reporting Time is fixed and static (HH:MM AM/PM), EXCEPT when a patient receives a demotion strike!
6. Cascading Re-Reporting Demotion Protocol:
   - 0 to 5m late: Grace Period ("⚠️ Late Xm", zero penalty).
   - Past 5m late: Strike 1 ("⚠️ Strike 1 (-2 Slots)", demote +2 slots down, recalculate Reporting Time for new position).
   - Past 5m of new deadline: Strike 2 ("🛑 Strike 2 (-5 Slots)", demote +5 slots down, recalculate Reporting Time again).
   - Past 5m of 3rd deadline: Strike 3 ("❌ Strike 3 (Cancelled)", evict to last slot of day / Cancelled).
7. receptionist.html UI:
   - Header with HH:MM clock, doctor sync widget, speed controls (1x, 5x, 15x, Pause, Reset 20 Patients).
   - Active Cabin Hero Card at top of desk showing active patient details, entry timestamp ("Entered Cabin at HH:MM AM"), and live MM:SS ticking timer.
   - Master Queue Table with 8 columns (Pos, Token, Patient Name + Strike Badges, Category Badge, Reporting Time, Estimated Turn, Gate Status, More Options).
   - Combined Gate Status column (📍 ADMIT button for en-route, ✓ LOBBY tag for checked-in, HOLD or LAB tag with 🏃 Return).
   - Far-right More Options (⋮) dropdown menu (HOLD, Print Slip, WhatsApp QR, Manual Demotion).
8. doctor.html UI:
   - Doctor profile header (Dr. Sarah Sharma, Cabin 1, Shift 09:00 - 12:00) with Baseline ACT, Rolling ACT, Seen Today count, and top HH:MM clock.
   - "I'M IN (CLOCK IN)" button guard preventing patient calling until doctor arrives.
   - Giant Active Consultation Focus Hero with MM:SS timer.
   - Buttons: "🔔 FINISH VISIT & CALL NEXT PATIENT", "🧪 Refer to Lab / X-Ray", "☕ Schedule Break" (5m, 10m, 15m modal), "⏰ Report Delay".
   - Right Buffer Panel rendering verified lobby arrivals waiting outside Cabin 1.

Please generate complete, production-ready, fully functional code for both files without placeholders.
```

---

## 📑 PART 2: COMPLETE TECHNICAL REFERENCE NOTES

### 1. SYSTEM PURPOSE & CORE CONCEPTS

The **Velora OPD Engine** is a dual-dashboard, real-time hospital Outpatient Department queue intelligence system. It solves three critical hospital OPD problems:
1. **Waiting Room Overcrowding**: Patients are given a **Static Reporting Time** (arrival deadline) based on a **30-Minute Maximum Lead Window**, ensuring they arrive just-in-time rather than waiting hours in the lobby.
2. **Doctor Pace Variation**: A dynamic **3-Consecutive Patient Sample Rolling ACT Engine** smoothly adjusts estimated turn times based on actual doctor consultation speed without skewing from one-off fast or slow visits.
3. **No-Show & Late Patient Penalties**: A **Cascading Re-Reporting Demotion Protocol** automatically penalizes late arrivals by demoting them down the queue, promoting present patients, and recalculating arrival deadlines for demoted slots.

---

### 2. SHARED TELEMETRY STATE SCHEMA (JSON)

Both dashboards synchronize in real time via HTML5 LocalStorage under the key `VELORA_OPD_SHARED_STATE_V7` at 100ms polling intervals:

```json
{
  "simBaseTime": 31200,
  "realBaseTimestamp": 1771910000000,
  "isRunning": true,
  "speedMultiplier": 1,
  "doctorState": {
    "isClockedIn": false,
    "cabinStatus": "NOT_CLOCKED_IN",
    "activePatient": null,
    "consultStartSimSec": 0,
    "rollingACT": 10.0,
    "fastStreakCount": 0,
    "slowStreakCount": 0,
    "completedCount": 0,
    "pendingBreakMins": 0,
    "breakRemainingSec": 0,
    "breakStartSimSec": 0,
    "delayMins": 0,
    "pendingVIPAlert": null
  },
  "patients": [
    {
      "token": "P-1",
      "name": "Rajesh Kumar",
      "type": "APP",
      "reportingSec": 31200,
      "etaSec": 32400,
      "status": "EN_ROUTE",
      "strike": 0,
      "prio": "REGULAR"
    }
  ],
  "walkinCounter": 1,
  "emergencyCounter": 1,
  "vipCounter": 1,
  "lastUpdated": 1771910000000
}
```

---

### 3. QUEUE PRIORITY RANKING & INSERTION RULES

The queue table is strictly sorted by **Priority Hierarchy**, then by **Estimated Turn Time**:

| Rank | Category | Token Prefix | UI Badge Style | Insertion & Priority Logic |
| :--- | :--- | :--- | :--- | :--- |
| **#1** | **Emergency** | `E-1`, `E-2` | Red Pulsing Pill (`🚨 EMERGENCY`) | Inserted at **Position #1** ahead of regular patients. Allocated $1.5 \times \text{ACT}$ (15m). |
| **#2** | **VIP Reference** | `V-1`, `V-2` | Gold Pill (`👑 VIP REF`) | Inserted behind emergencies, ahead of regular queue. Triggers instant doctor cabin alert banner. Allocated $1.5 \times \text{ACT}$ (15m). |
| **#3** | **Lab Return** | `P-X` | Cyan Badge (`🔬 Lab Return`) | Re-admitted to **Position #2** after radiology/lab report pickup (2-3m review). |
| **#4** | **Restroom Return** | `P-X` | Purple Badge (`🏃 Return`) | Restored to original pre-hold slot upon returning to lobby. |
| **#5** | **App Booking** | `P-1`..`P-20` | Cyan Pill (`📱 App`) | Standard FIFO appointment slot sequence. Allocated $1.0 \times \text{ACT}$ (10m). |
| **#6** | **Walk-In** | `W-1`..`W-N` | Blue Pill (`📋 Walk-In`) | Appended to next available slot in queue. Allocated $1.0 \times \text{ACT}$ (10m). |

---

### 4. AVERAGE CONSULTATION TIME (ACT) ENGINE

#### 🔹 Baseline & Clamping Limits
* Baseline ACT: **`10.0 minutes`**.
* Clamping Limits: Minimum **`4.0 minutes`**, Maximum **`30.0 minutes`**.

#### 🔹 Exponential Moving Average (EMA) Formula
When a standard-pace consultation finishes:

$$\text{Rolling ACT}_{\text{new}} = (0.7 \times \text{Rolling ACT}_{\text{old}}) + (0.3 \times \text{Actual Duration})$$

#### 🔹 The 3-Consecutive Patient Sample Rule & Thresholds
1. **Threshold Definitions**:
   * **Fast Threshold**: $\le 0.5 \times \text{Rolling ACT}$ (e.g. $\le 5$ mins when ACT = 10m).
   * **Slow Threshold**: $\ge 1.5 \times \text{Rolling ACT}$ (e.g. $\ge 15$ mins when ACT = 10m).
2. **Single Patient Anomaly Guard**:
   * If only 1 or 2 visits finish fast or slow, it is treated as a temporary anomaly. **Rolling ACT does NOT shift.**
3. **3 Consecutive Sample Trigger**:
   * **3 Fast Visits ($\le 0.5 \times \text{ACT}$)**:
     $$\text{Shifted ACT} = (0.6 \times \text{ACT}_{\text{old}}) + (0.4 \times \text{Fast Threshold})$$
     *Result*: Advances all downstream Estimated Turns earlier!
   * **3 Slow Visits ($\ge 1.5 \times \text{ACT}$)**:
     $$\text{Shifted ACT} = (0.6 \times \text{ACT}_{\text{old}}) + (0.4 \times \text{Slow Threshold})$$
     *Result*: Extends downstream Estimated Turns and pushes arrival times!

#### 🔹 Four System Exemptions (Excluded from ACT Average)
1. **Emergency & VIP Normalization**: Actual consultation duration is divided by 1.5 ($\text{duration} \div 1.5$) before being evaluated by ACT logic.
2. **Diagnostic Lab Returns**: Quick 2-3 minute report review visits are **100% excluded** from the moving average.
3. **Restroom Holds**: Restroom returns are **100% excluded**.
4. **Doctor Coffee Breaks & Delays**: Rest breaks (5m, 10m, 15m) and reported delays shift the queue timeline cursor, but are **100% excluded** from ACT.

---

### 5. STATIC REPORTING TIME & 30-MINUTE LEAD CAPPING MATH

#### 🔹 Formulas
$$\text{Uncapped Lead Buffer} = 2.0 \times \text{Rolling ACT}$$

$$\mathbf{\text{Lead Buffer}} = \text{clamp}(\text{Uncapped Lead Buffer}, \mathbf{5\text{ Mins Minimum}}, \mathbf{30\text{ Mins Maximum}})$$

$$\mathbf{\text{Reporting Time}} = \text{Estimated Turn Time} - \text{Lead Buffer}$$

* **Guaranteed Constraint**: Reporting Time is **ALWAYS strictly $\le$ Estimated Turn Time $- 5$ Minutes**, ensuring every patient has at least 5 minutes to check in before their consultation turn!
* **Unique Sequential Timelines**: Every patient is assigned a distinct, non-overlapping Estimated Turn Time spaced apart by ACT duration.

#### 🔹 Static Reporting Time Rule
* A patient's calculated **Reporting Time** is stamped upon appointment booking (e.g. `08:40 AM` for a `09:00 AM` turn).
* It remains **fixed and static** (`HH:MM AM/PM`). It does **not** tick forward with simulation clock seconds.
* **Sole Exemption**: A patient's Reporting Time changes **ONLY** when they miss a reporting deadline, receive a demotion strike, and are shifted to a new queue position!

---

### 6. CASCADING RE-REPORTING DEMOTION PROTOCOL

When the simulation clock passes a patient's **Reporting Time** (e.g. `08:40 AM`) while their status is still `EN_ROUTE` (`AWAY`):

```
                                  [08:40 AM Deadline]
                                           │
                        ┌──────────────────┴──────────────────┐
                        ▼                                     ▼
             [Checked-In ≤ 08:40 AM]              [Not Checked-In at 08:40 AM]
                        │                                     │
               🟢 Normal Lobby Flow                 08:40 AM to 08:45 AM
              (Retains Original Pos)               ⚠️ Grace Period ("Late 3m")
                                                              │
                                                     Past 08:45 AM (5m+ Late)
                                                              │
                                                    🛑 STRIKE 1 DEMOTION
                                                    - Demoted +2 Slots Down (Pos #1 ➔ Pos #3)
                                                    - NEW Reporting Time Calculated for Pos #3 (08:50 AM)
                                                              │
                                                     Past New Deadline + 5m
                                                              │
                                                    🛑 STRIKE 2 DEMOTION
                                                    - Demoted +5 Slots Down
                                                    - NEW Reporting Time Calculated again
                                                              │
                                                     Past Final Deadline + 5m
                                                              │
                                                    ❌ STRIKE 3 EVICTION
                                                    - Moved to LAST SLOT of the day / Cancelled
```

#### 📋 Step-by-Step Strike Breakdown:
1. **Grace Period (`08:40 AM – 08:45 AM`)**: Retains Pos #1, shows yellow badge `⚠️ Late 3m`. Zero penalty if checked in during grace.
2. **Strike 1 (At `08:45 AM`)**: Demoted **+2 slots down** among ON-TIME / NON-DEMOTED patients (Pos #1 $\rightarrow$ Pos #3). Displays a single **Tiny Yellow Dot (`🟡`)** next to name. **Reporting Time is recalculated for Pos #3 (`08:50 AM`)**!
3. **Strike 2 (At `08:55 AM`)**: Demoted **+5 slots down** among ON-TIME / NON-DEMOTED patients. Displays **Yellow + Red Dots (`🟡🔴`)** next to name. Reporting Time is recalculated again for new slot!
4. **Strike 3 (Final Eviction)**: Displays **Yellow + Red + Slate Dots (`🟡🔴⚫`)**. Moved to **very last slot of day / Cancelled**.

---

### 7. RECEPTIONIST DESK PORTAL (`receptionist.html`) UI SPECIFICATIONS

1. **Header Bar**: Simulation clock (`HH:MM AM/PM`), Doctor Sync Telemetry (`🔴 Away` / `🟢 Consulting` / `☕ On Break`), Speed controls (`1x`, `5x`, `15x`, `Pause`, `Reset 20 Patients`).
2. **Left Panel (Registration & Quick Triggers)**: Walk-in registration form, Category dropdown, Quick Triggers (`🚨 Emergency Arrival`, `🔬 Diagnostic Lab Return`).
3. **Active Cabin Hero Card**: Top card showing active token, patient name, type badge, entry timestamp (*"Entered Cabin at 09:00 AM"*), and live `MM:SS` ticking timer.
4. **Master Queue Table**: 8 columns (`Pos`, `Token`, `Patient Name`, `Category`, `Reporting Time`, `Estimated Turn`, `Gate Status`, `More Options`).
5. **Combined Gate Status Column**: `📍 ADMIT` button for en-route, `✓ LOBBY` for checked-in, `HOLD` or `LAB` with `🏃 Return`.
6. **More Options (`⋮`) Dropdown**: Far-right dropdown (`HOLD`, `Print Slip`, `WhatsApp QR`, `Manual Demotion`).

---

### 8. DOCTOR CABIN CONSOLE (`doctor.html`) UI SPECIFICATIONS

1. **Header Bar**: Doctor profile (Dr. Sarah Sharma, Cabin 1, Shift 09:00 - 12:00), Baseline ACT (`10.0m`), Rolling ACT (`10.0m`), Patients Seen (`0`), Top clock (`HH:MM AM/PM`).
2. **"I'M IN" Doctor Clock-In Guard**: Primary button shows `🚪 I'M IN (CLOCK IN TO CABIN 1)`. Calling disabled until doctor arrives.
3. **Active Focus Hero**: Giant active token badge, patient name, category, large `MM:SS` consultation clock display.
4. **Doctor Actions**:
   * `🔔 FINISH VISIT & CALL NEXT PATIENT`: Finish visit, trigger 3-sample Rolling ACT, admit next verified lobby patient.
   * `🧪 Refer to Lab / X-Ray`: Move active patient to diagnostic lab.
   * `☕ Schedule Break`: Modal to select 5m, 10m, or 15m break.
   * `⏰ Report Delay`: Broadcast doctor delay.
5. **Right Buffer Panel**: Renders verified lobby arrivals waiting outside Cabin 1.

---

### 9. STEP-BY-STEP SCENARIO EXECUTION TRACE

```
[08:40 AM] Simulation Starts
 ├── P-1 Rajesh Kumar (Reporting: 08:40 AM, ETA: 09:00 AM, Status: EN_ROUTE)
 ├── P-2 Ananya Roy   (Reporting: 08:50 AM, ETA: 09:10 AM, Status: EN_ROUTE)
 └── P-3 Suresh Patel (Reporting: 09:00 AM, ETA: 09:20 AM, Status: EN_ROUTE)

[08:42 AM] Clock Ticks Forward (Speed 5x)
 └── P-1 missed 08:40 AM check-in -> Row shows "⚠️ Late 2m" warning badge (Grace Window active).

[08:45 AM] 5-Minute Grace Window Expires
 ├── P-1 receives STRIKE 1!
 ├── P-1 is demoted +2 slots down (Pos #1 -> Pos #3).
 ├── P-2 Ananya Roy is promoted to Pos #1!
 └── P-1 Reporting Time is RE-CALCULATED for Pos #3 -> New Reporting Time: 08:50 AM!

[08:48 AM] P-1 Arrives at Reception Desk
 ├── Receptionist clicks "📍 ADMIT" on P-1.
 └── P-1 status changes to "✓ LOBBY". P-1 waits in lobby behind P-2.

[09:00 AM] Doctor Arrives & Clicks "I'M IN"
 ├── Dr. Sarah Sharma arrives in Cabin 1 and clicks "I'M IN".
 ├── Doctor clicks "🔔 CALL NEXT PATIENT".
 └── P-2 Ananya Roy (Pos #1, present in lobby) enters Cabin 1!
     Hero Card displays: "P-2 Ananya Roy • Entered Cabin at 09:00 AM • Timer: 00:01".
```
