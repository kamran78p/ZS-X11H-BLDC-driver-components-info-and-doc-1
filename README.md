# ZS-X11H-BLDC-driver-components-info-and-doc-1
Information and components for BLDC motor drivers
Complete Working Chain: JY01 → EG2134 → MOSFETs → Motor
1. Signal Mapping — Exact Pin Connections
JY01 (16‑pin Controller) → EG2134 (TSSOP‑20 Driver)
Table
JY01 Pin	Signal	EG2134 Pin	EG2134 Function
7	AT (Phase‑A High)	1 → HIN1	High‑side A input
8	AB (Phase‑A Low)	4 → LIN1	Low‑side A input
9	BT (Phase‑B High)	2 → HIN2	High‑side B input
10	BB (Phase‑B Low)	5 → LIN2	Low‑side B input
11	CT (Phase‑C High)	3 → HIN3	High‑side C input
12	CB (Phase‑C Low)	6 → LIN3	Low‑side C input
—	+12 V supply	7 → VCC	Driver power (4.5–20 V, typically 12 V)
3 (JY01)	GND	8 → GND	Common ground
EG2134 → MOSFET Gate Drive Outputs
Table
EG2134 Pin	Output	Connects To	MOSFET Type
19	HO1	Phase‑A High‑Side Gate	SSF7509 (N‑Ch High‑Side)
11	LO1	Phase‑A Low‑Side Gate	SSF7508 (N‑Ch Low‑Side)
16	HO2	Phase‑B High‑Side Gate	SSF7509
10	LO2	Phase‑B Low‑Side Gate	SSF7508
13	HO3	Phase‑C High‑Side Gate	SSF7509
9	LO3	Phase‑C Low‑Side Gate	SSF7508
Bootstrap Floating Power Per Phase (Critical for High‑Side N‑Channel!)
Table
EG2134 Pin	Name	Function
20 / 18 / 14	VB1 / VB2 / VB3	Bootstrap positive — ~12 V above phase voltage
18 / 15 / 12	VS1 / VS2 / VS3	Phase node — between high‑side & low‑side MOSFETs → Motor terminal
Each VB–VS pair uses a 100 nF–1 µF ceramic cap + fast‑recovery diode (FR107) to pump the 12 V supply UP to follow the phase rail — so when the high‑side MOSFET turns ON, its gate gets ~12 V above the battery voltage, fully enhancing the N‑channel.
2. EG2134 Internal Logic — How It Decides
From the datasheet truth table — this is the safety core:
Table
HINx	LINx	HOx	LOx	Status
0	0	OFF	OFF	Safe idle
1	0	ON	OFF	✅ High‑side ON
0	1	OFF	ON	✅ Low‑side ON
1	1	OFF	OFF	⛔ Lockout — both forced OFF
Key Points:
JY01 never sends 1+1 together — it follows the valid BLDC sequence
If JY01 glitches or wiring errors happen → EG2134 automatically blocks both → no shoot‑through
Dead‑time: 50–300 ns built‑in → one device fully off before the other turns on
Floating inputs = internal 240 kΩ pull‑down → both outputs OFF — failsafe
3. MOSFET Roles — SSF7508 vs SSF7509
Both are N‑Channel Enhancement Mode Power MOSFETs — meaning:
Vgate > Vsource + ~10 V → fully ON (low resistance)
Vgate = Vsource → fully OFF
Table
Part	Position	Vds	Id @25°C	Rds(on)	Source Connection
SSF7508	Low‑Side	75 V	40 A	12.5 mΩ → higher	Source → GND / R005 shunt
SSF7509	High‑Side	75 V	80 A	6.5 mΩ → lower	Source → VSx / motor phase
Why different parts?
Low‑Side (SSF7508): source is fixed at GND → gate drive is steady 0 V ↔ 12 V — simple
High‑Side (SSF7509): source floats — when ON, it sits at +Battery (e.g., 48 V) → gate must be at 48 V + 12 V = 60 V → that’s what the bootstrap VB–VS network creates
SSF7509 has lower Rds(on) → less heat at high current → used on high‑side where conduction losses dominate
4. Full Operating Sequence — Step by Step
Step 1 — Power Up
plaintext
Battery + → XL7005 buck → ~12 V → EG2134 VCC (pin 7)
                          ↓
                       78L05 → +5 V → JY01 Vdd (pin 4) + Hall sensors + potentiometer
JY01 powers up, waits for stable Hall signals
EG2134 VCC = 12 V → ready to drive gates with 12 V swing
Step 2 — Position Sensing
Motor’s 3 Hall sensors → Ha/Hb/Hc → JY01 pins 13/14/15
JY01 decodes 6 valid rotor positions (120° spacing)
For each position, JY01 outputs exactly one of the 6 valid drive patterns:
Step 3 — One Commutation State Example
State: Phase A High, Phase B Low, Phase C Off
Table
JY01 Output	Value	EG2134 Action	MOSFET Result
AT (HIN1)	1	HO1 = 12 V → SSF7509(A) gate ON	A connected to +BATT ✅
AB (LIN1)	0	LO1 = 0 V → SSF7508(A) gate OFF	—
BT (HIN2)	0	HO2 = 0 V → SSF7509(B) gate OFF	—
BB (LIN2)	1	LO2 = 12 V → SSF7508(B) gate ON	B connected to GND ✅
CT (HIN3)	0	HO3 = 0 V → SSF7509(C) gate OFF	—
CB (LIN3)	0	LO3 = 0 V → SSF7508(C) gate OFF	—
Current path:
plaintext
+Battery → SSF7509(A) → Motor Winding A → Winding B → SSF7508(B) → R005 → GND
Magnetic field pulls rotor forward → Hall pattern changes → JY01 moves to next state
Step 4 — Speed Control via PWM
Potentiometer → JY01 pin 16 (VR) → sets duty cycle
JY01 rapidly switches the high‑side signal ON/OFF (PWM) while keeping the commutation pattern correct
EG2134 faithfully amplifies these fast pulses → MOSFETs switch at the same rate
PWM ON → full voltage applied; PWM OFF → winding freewheels through diode
Average voltage = duty cycle × battery voltage → controls motor RPM
Step 5 — Bootstrap in Action (The Magic)
When HO1 turns ON:
SSF7509(A) conducts → VS1 (pin 18) jumps from 0 V → ~48 V
Capacitor across VB1–VS1 was previously charged to 12 V
Now VB1 automatically jumps to 48 V + 12 V = 60 V
HO1 follows → gate sits at 60 V → SSF7509 stays fully ON
During LO1 ON periods → capacitor recharges through diode → ready for next cycle
Step 6 — Protection Layers
Table
Layer	Where	What Happens
Over‑Current	R005 → JY01 pin 2 (IS)	>20 A → voltage >0.1 V → JY01 shuts down all outputs
Cross‑Conduction	EG2134 internal	If HIN & LIN both HIGH → both OFF instantly
Dead‑Time	EG2134 internal 50–300 ns	Prevents overlap during switching transitions
Under‑Voltage	EG2134 VCC < ~4.5 V	Outputs disabled → MOSFETs safe
Locked Rotor	JY01 firmware	No Hall change → ~3 s → shutdown; reduce VR to reset
Step 7 — Braking
JY01 pin 1 pulled LOW → all LINx = 1, all HINx = 0
EG2134 → all LOx = ON, all HOx = OFF
All 3 low‑side SSF7508s ON → motor windings shorted together → heavy braking current → energy dissipated in motor windings & R005
5. Summary Diagram — Signal Flow
plaintext
Hall Sensors → JY01 ←── Speed Pot/DIR/BRAKE
                  │
                  │ 6 logic lines (AT/AB/BT/BB/CT/CB)
                  ▼
            ┌──────────────┐
            │   EG2134     │
            │ 3× Half-Bridge Driver │
            │ Dead-Time + Bootstrap │
            └───┬───┬───┬───┘
                │   │   │
          HO1─┐ │ HO2─┐ │ HO3─┐
          LO1 │ │ LO2 │ │ LO3 │
              ▼   ▼   ▼   ▼   ▼   ▼
           SSF7509  SSF7509  SSF7509   ← High-Side N-Channels
              │        │        │
             ─┼────────┼────────┼─────── Motor Phases A/B/C
              │        │        │
           SSF7508  SSF7508  SSF7508   ← Low-Side N-Channels
              │        │        │
              ┴────────┴────────┴── R005 ── GND
6. Why This Combo Works So Well Together
JY01 = the brain — knows the sequence, speed, direction, protection
EG2134 = the muscle amplifier — takes weak 5 V logic → strong 12 V gate drive with built‑in safety
SSF7508/7509 = the heavy lifters — handle high current efficiently
Bootstrap = the clever trick — lets simple N‑channel MOSFETs work on the high‑side without expensive isolated supplies
Dead‑time + lockout = the safety net — even if software glitches, hardware prevents destruction
