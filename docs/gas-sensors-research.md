# SANITRAC TMS — Gas Sensor Research

**Board:** SANITRAC TMS‑V1.2 (25/5/2026) — Top Module
**MCU:** ESP32‑S3 (WROOM‑1 module)
**Gas sensors fitted:** MQ‑2, MQ‑4, MQ‑9, MQ‑137
**Deployment context (from cost analysis):** Nashik Trimbakeshwar Simhastha Kumbh 2027 — mass‑gathering public toilets
**Status:** research / specification. No firmware written yet.

---

## 0. How to read this document

Everything in §3 (per‑sensor specifications) is taken from the **primary manufacturer datasheets**, which
I downloaded and text‑extracted. Those numbers are reliable.

Everything in §5 (expected readings) and §7 (thresholds) is **engineering judgement** built on published
exposure limits and measured restroom studies. They are good starting points, not calibrated truth.

The ppm curve coefficients in §6 come from widely‑used derived libraries, **not** from the datasheets —
the datasheets only publish the curves as images. I have flagged where those coefficients disagree with
the datasheet's own specifications. Treat them as a starting point that must be replaced by per‑unit
calibration.

---

## 1. The honest headline

Three things you should know before any code gets written.

**1. MQ‑137 is the only sensor here that is actually aimed at toilet air.** NH₃ from urea breakdown is the
dominant toilet odorant, and MQ‑137 is the NH₃ part. MQ‑2, MQ‑4 and MQ‑9 are combustible‑gas and CO parts.
In a toilet with no gas leak and no smoker, they will sit at baseline essentially all the time.

**2. There is no H₂S sensor in the BOM, and H₂S is the gas people actually complain about.** Hydrogen
sulphide (rotten‑egg) has an odour threshold around **0.0005–0.3 ppm** — humans smell it roughly a
thousand times lower than ammonia. Measured dry‑sanitation toilets showed **0.22 and 3.17 ppm H₂S**
alongside 3.16 and 6.88 ppm NH₃. The part you would want is **MQ‑136** (H₂S). None of MQ‑2/4/9/137
specifies an H₂S response. This is the single biggest gap in the sensor set — see §9.

**3. Your worst‑case toilet is near the *bottom* of MQ‑137's range, not the middle.** MQ‑137 is specified
from **5 ppm** NH₃ upward. A clean, ventilated toilet is well under 1 ppm; a bad one is 6–7 ppm; the
often‑quoted "urine smell in public toilets" figure is 10–40 ppm. So the interesting part of your signal
sits at or below the sensor's specified floor. **The system must therefore work on deviation from a
learned per‑unit baseline, not on absolute ppm.** This is the most important design decision in the whole
gas subsystem, and §8 is built around it.

---

## 2. What is actually in toilet air

| Gas | Source in a toilet | Human odour threshold | Typical clean | Typical foul | Covered by your BOM? |
|---|---|---|---|---|---|
| **Ammonia (NH₃)** | Urea hydrolysis by urease bacteria in urine | ~5 ppm (pungent well before that for some) | < 1 ppm | 3–7 ppm measured; 10–40 ppm reported for heavy urine smell | ✅ **MQ‑137** |
| **Hydrogen sulphide (H₂S)** | Faeces, anaerobic sludge, sewer/trap seal loss | **0.0005–0.3 ppm** | < 0.01 ppm | 0.2–3.2 ppm measured | ❌ **no sensor** (needs MQ‑136) |
| **Methyl mercaptan, indole, skatole, volatile fatty acids** | Faecal matter | sub‑ppb to ppb | trace | trace–ppm | ⚠️ partial, non‑specific (MQ‑2 as generic VOC) |
| **Methane (CH₄)** | Sewer gas, septic tank, blocked trap | odourless | ~2 ppm (ambient) | 100s–1000s ppm on sewer gas ingress | ✅ **MQ‑4** (and MQ‑9) |
| **CO₂** | Occupancy / breathing | odourless | 400–600 ppm | 1500–3000 ppm (poor ventilation) | ❌ no sensor (NDIR needed; MQ series cannot) |
| **CO** | Only if combustion nearby — generator, gas geyser, diesel exhaust drawn in | odourless | < 1 ppm | 9 ppm+ is already an IAQ concern | ⚠️ **MQ‑9**, but only with heater cycling (§3.3) |
| **LPG / butane / propane** | Aerosol air‑freshener propellant, lighters | — | ~0 | 100s ppm transient during a spray | ✅ **MQ‑2**, MQ‑9 |
| **Ethanol / IPA / quaternary ammonium / chlorine** | Cleaning chemicals, sanitiser | — | ~0 | high transient during cleaning | ✅ **MQ‑2** (strong alcohol response) |
| **Tobacco smoke** | Smoking in cubicles | — | 0 | — | ✅ **MQ‑2**, and MQ‑137 responds too (§4) |

**Placement note:** NH₃ (MW 17) and CH₄ (MW 16) are lighter than air and rise; H₂S (MW 34) is heavier and
pools low. Your gas sensors are on the **Top Module**, which is the correct place for NH₃ and CH₄ — and
another reason an H₂S channel would need separate thought about mounting height.

---

## 3. Per‑sensor specifications (from primary datasheets)

All four are SnO₂ (tin dioxide) semiconductor sensors. Resistance **falls** as reducing‑gas concentration
rises. All need a heater, all drift, all are cross‑sensitive, none is a selective analytical instrument.

### 3.1 MQ‑137 — Ammonia (the important one)

| Parameter | Value |
|---|---|
| Detection gas | Ammonia, and other organic amines |
| **Detection range** | **5 – 500 ppm NH₃** |
| Loop voltage V_C | ≤ 24 V DC (use 5.0 V ± 0.1 V) |
| Heater voltage V_H | 5.0 V ± 0.2 V, AC or DC |
| Heater resistance R_H | 31 Ω ± 3 Ω (room temp) |
| **Heater consumption P_H** | **≤ 900 mW** (≈ 180 mA @ 5 V) |
| Load resistance R_L | Adjustable |
| Sensing resistance R_s | **2 kΩ – 15 kΩ in 50 ppm NH₃** |
| Slope α | ≤ 0.6 (R₁₀₀ppm / R₅₀ppm NH₃) |
| Standard test conditions | 20 °C ± 2 °C, 65 % ± 5 % RH |
| **Preheat time** | **Over 48 hours** |
| R₀ reference (Fig. 1) | sensor resistance in **50 ppm ethanol** |
| R₀ reference (Fig. 2, temp/humidity) | resistance in 50 ppm NH₃ at 20 °C / 65 % RH |

**Datasheet quirks worth knowing:**

- The sensitivity spec line reads `Rs(in air)/Rs(5000ppm CH4) ≥ 5` — that is a copy‑paste error from the
  MQ‑2 datasheet. Read it as the NH₃ equivalent.
- The R₀ reference gas for the sensitivity curve is **50 ppm ethanol**, not clean air. This is why
  library "clean air ratio" values for MQ‑137 are inconsistent between sources (§6.2).
- **Smoke note, directly from the datasheet:** *"Sensitivity to smoke is ignite 10pcs cigarettes in 8 m³
  room, and the output equals to 10 ppm NH₃."* An 8 m³ room is about the size of a toilet cubicle. So
  **one or two cigarettes in a cubicle will look like a several‑ppm ammonia event.** See §4.

**α ≤ 0.6 sanity check:** a ratio of ≤0.6 per doubling of concentration implies a log‑log slope of
|b| ≥ 0.737. Keep this number — §6.2 uses it to show that the commonly circulated library coefficients
for MQ‑137 cannot be right.

### 3.2 MQ‑2 — Combustible gas & smoke

| Parameter | Value |
|---|---|
| Detection gas | Combustible gas and smoke; high sensitivity to LPG, propane, hydrogen |
| **Detection range** | **300 – 10 000 ppm** (combustible gas) |
| Loop voltage V_C | ≤ 24 V DC (use 5.0 V ± 0.1 V) |
| Heater voltage V_H | 5.0 V ± 0.2 V |
| Heater resistance R_H | 31 Ω ± 3 Ω |
| **Heater consumption P_H** | **≤ 900 mW** (≈ 180 mA @ 5 V) |
| Load resistance R_L | Adjustable |
| Sensing resistance R_s | 2 kΩ – 20 kΩ in 2000 ppm C₃H₈ |
| Sensitivity S | Rs(air) / Rs(1000 ppm isobutane) ≥ 5 |
| Slope α | ≤ 0.6 (R₅₀₀₀ppm / R₃₀₀₀ppm CH₄) |
| **Preheat time** | **Over 48 hours** |
| R₀ reference | sensor resistance in **1000 ppm hydrogen** |
| Temp/humidity curve (Fig. 2) | plotted at 30 %, 60 %, 85 % RH, −20 °C to +50 °C; Rs/R₀ spans ≈ 0.5 – 1.9 |

Note the datasheet gives `Rs = (Vc/VRL − 1) × RL` explicitly, and
`Ps = Vc² × Rs / (Rs + RL)²` for sensing‑body power.

### 3.3 MQ‑9 — CO / combustible gas (⚠️ needs a switched heater)

| Parameter | Value |
|---|---|
| Circuit voltage V_C | 5 V ± 0.1 |
| **Heating voltage HIGH V_H(H)** | **5 V ± 0.1 for 60 ± 1 s** |
| **Heating voltage LOW V_H(L)** | **1.4 V ± 0.1 for 90 ± 1 s** |
| Heater resistance R_H | 33 Ω ± 5 % |
| **Heater consumption P_S** | **< 340 mW** |
| Load resistance R_L | Adjustable (10 kΩ ± 5 % used for the datasheet curve) |
| Sensing resistance R_s | 2 – 20 kΩ in 100 ppm CO |
| Slope α | < 0.5 (Rs 300 ppm / Rs 100 ppm) |
| **Detection range** | **20 – 2000 ppm CO**; 500 – 10 000 ppm CH₄; 500 – 10 000 ppm LPG |
| **Preheat time** | **No less than 48 hours** |
| R₀ reference | sensor resistance in **1000 ppm LPG** in clean air |
| Standard conditions | 20 °C ± 2 °C, 65 % ± 5 % RH |

**This is the one real design trap.** MQ‑9 works by cycling temperature: at **1.4 V (low temperature) it
measures CO**; at **5 V (high temperature) it measures methane/LPG and burns off gases adsorbed during the
low phase**. Reading CO from an MQ‑9 whose heater is permanently tied to 5 V produces a number, but that
number is not CO.

**Action required:** confirm from the schematic whether the TMS‑V1.2 board can switch the MQ‑9 heater
between 5 V and 1.4 V (a MOSFET/transistor driven from a GPIO, plus a resistor divider or a second
regulator). Looking at the board photos I cannot see a per‑sensor heater driver, which suggests the
heaters are all hard‑tied to +5 V.

- **If the heater is switchable:** implement the 60 s / 90 s cycle, sample CO in the last ~10 s of the low
  phase and combustibles in the last ~10 s of the high phase. Cadence is then one reading per 150 s.
- **If the heater is fixed at 5 V:** do **not** publish a CO value from MQ‑9. Use it as a second
  combustible‑gas channel and cross‑check for MQ‑4. Say so explicitly in the API/dashboard rather than
  shipping a fake CO number to a public‑health dashboard.

### 3.4 MQ‑4 — Methane / natural gas

| Parameter | Value |
|---|---|
| Circuit voltage V_C | 5 V ± 0.1 |
| Heating voltage V_H | 5 V ± 0.1 |
| **Load resistance** | **20 kΩ** (recommended 10 kΩ – 47 kΩ) |
| Heater resistance R_H | 33 Ω ± 5 % |
| **Heater consumption P_H** | **< 750 mW** (≈ 150 mA @ 5 V) |
| Sensing resistance R_s | 10 kΩ – 60 kΩ in 1000 ppm CH₄ |
| Slope α | ≤ 0.6 (1000 ppm / 5000 ppm CH₄) |
| **Detection range** | **200 – 10 000 ppm CH₄ / natural gas** |
| **Preheat time** | **Over 24 hours** |
| Operating temp | −10 °C to +50 °C |
| Humidity | < 95 % RH |
| O₂ | 21 % standard; **minimum 2 %** — O₂ concentration affects sensitivity |
| R₀ reference | sensor resistance at **1000 ppm CH₄** in clean air |
| Calibration advice (datasheet) | calibrate at **5000 ppm CH₄** with R_L ≈ 20 kΩ |

Datasheet explicitly notes **small sensitivity to alcohol and smoke** — that is a feature here: it makes
MQ‑4 a cleaner methane channel than MQ‑2, and a useful discriminator against cleaning‑chemical events.

### 3.5 Common environmental limits (all four)

- Operating temperature: roughly −10/−20 °C to +50 °C. Fine for Nashik.
- Humidity: < 95 % RH, **no condensation**. A washed‑down toilet block will get close to this. Condensation
  on the sensing element permanently reduces sensitivity.
- **All four datasheets warn: exposure to H₂S, SO_X, Cl₂, HCl "will not only result in corrosion of sensors
  structure, also it cause sincere sensitivity attenuation."** You are deliberately installing these in an
  environment that has H₂S and chlorine‑based cleaning products. Expect accelerated drift and plan
  replacement (§9).
- Silicone vapour permanently poisons SnO₂ sensors. Do not use silicone sealant, silicone‑bearing potting,
  or silicone gaskets anywhere in the enclosure. Check the enclosure gasket material.

---

## 4. Cross‑sensitivity — and how to turn it into an advantage

SnO₂ sensors are not selective, and in a naive implementation this is a source of false alarms. With four
of them you can instead use the *pattern* across channels to classify events. This is the most valuable
thing the four‑sensor array can do that a single sensor cannot.

| Event | MQ‑137 (NH₃) | MQ‑2 (combustible/smoke) | MQ‑4 (CH₄) | MQ‑9 (CO/comb.) | Signature |
|---|---|---|---|---|---|
| **Urine accumulating / not flushed** | slow steady rise over 10s of minutes | flat | flat | flat | MQ‑137 alone, slow ramp |
| **Faecal / sewer odour** | moderate rise | slight | **rise** (CH₄) | slight | MQ‑137 + MQ‑4 together |
| **Someone smoking** | **sharp rise** (datasheet: 10 cigarettes/8 m³ ≈ 10 ppm NH₃‑equivalent) | **sharp rise** | slight | rise | MQ‑2 **and** MQ‑137 spike together, fast edge |
| **Aerosol air freshener sprayed** | slight | **very sharp spike** (butane/propane propellant) | slight | sharp | MQ‑2 dominant, seconds‑long spike, fast decay |
| **Cleaning in progress** (alcohol/ammoniacal cleaner) | **rise** (ammoniacal cleaners contain NH₃!) | **rise** (alcohol) | flat | slight | MQ‑2 + MQ‑137, correlated with an RFID cleaner scan |
| **Sewer gas ingress / dry trap** | rise | rise | **strong rise** | **strong rise** | MQ‑4 + MQ‑9 lead |
| **Genuine LPG leak / fire risk** | flat | **strong rise** | rise | **strong rise** | MQ‑2 + MQ‑9 lead, MQ‑137 flat |

Two consequences for the firmware:

1. **Rate of change is as informative as level.** Urine odour ramps over tens of minutes; a spray or a
   cigarette is a step change with a fast decay. Track both the level and the first derivative.
2. **The RFID cleaner scan is a label.** When a cleaner badges in and MQ‑2 + MQ‑137 rise together, that is
   a *cleaning event*, not a *dirty toilet* — suppress the odour alarm for a hold‑off window and use the
   post‑cleaning settled baseline as a fresh reference. This is the cheapest possible way to get labelled
   training data out of a deployed fleet.

---

## 5. What values should each sensor actually show?

This is the core of your question. Two tables: raw electrical, then interpreted.

### 5.1 What you should see electrically (sanity checks for bring‑up)

Assuming V_C = 5.0 V and the sensor's own R_L, these are the checks that tell you the hardware works:

| Check | Expected | If not |
|---|---|---|
| Heater current per sensor at 5 V | MQ‑2 ≈ 160–180 mA, MQ‑137 ≈ 160–180 mA, MQ‑4 ≈ 150 mA, MQ‑9 ≈ 68 mA (5 V phase) | open heater, or wrong pin — see datasheet warning about applying voltage to pins 1‑3 / 4‑6 |
| Sensor body warm to touch after ~1 min | yes, distinctly warm | heater not powered |
| V_RL at power‑on (cold) | **high** — sensors read low resistance when cold, output starts high | — |
| V_RL after 5–20 min | falls and settles | still drifting fast = not warmed up |
| V_RL after 48 h burn‑in in clean air | stable within a few % over an hour | sensor damaged/poisoned, or supply sagging |
| Breathe on the sensor / hold an alcohol wipe near it | MQ‑2 V_RL jumps sharply, recovers in 10–60 s | no response = dead element |
| Hold a drop of household ammonia cleaner near MQ‑137 | strong response | dead element or wrong channel |

**Critical:** a "0" or a rail‑stuck reading is not a low‑gas reading, it is a fault. Detect and report
open/short channels explicitly rather than publishing 0 ppm.

### 5.2 What the numbers should mean (interpretation bands)

Anchors used: NIOSH REL, OSHA PEL, and measured restroom studies (sources in §11).

**MQ‑137 → NH₃ — your primary cleanliness channel**

| Band | NH₃ | What it means | Action |
|---|---|---|---|
| Clean | **< 1 ppm** (below sensor floor — read as *baseline*) | Freshly cleaned, ventilated | Status green |
| Normal in use | 1 – 5 ppm | Ordinary occupied toilet | Green |
| Noticeable odour | **5 – 15 ppm** | Urine accumulating; users will notice | Amber — schedule cleaning |
| Foul | **15 – 25 ppm** | Clearly unpleasant; complaint territory | **Dispatch cleaning now** |
| Occupational limit | **> 25 ppm** | NIOSH REL TWA = 25 ppm; OSHA 8‑h standard for ammonia commonly cited at 20–50 ppm | Red — force ventilation, escalate |
| Short‑term limit | **> 35 ppm** | NIOSH STEL = 35 ppm | Urgent — this is a staff‑exposure issue, not just odour |

Reference points: measured dry‑sanitation facilities showed **3.16 and 6.88 ppm** NH₃; "urine smell in
public toilets" is reported around **10–40 ppm**. So your amber/red bands are correctly placed for the
application.

**MQ‑2 → combustible / smoke — event channel**

| Band | Reading | Meaning | Action |
|---|---|---|---|
| Baseline | at learned R₀ | nothing happening | — |
| Transient spike, fast decay | any | aerosol spray / cleaning chemical | log as event, hold off odour alarms |
| Sustained rise + MQ‑137 rise, fast edge | any | **smoking in cubicle** | log/notify (relevant at a religious gathering site) |
| **> 1000 ppm LPG‑equivalent sustained** | ~2 % LEL of propane | genuine combustible presence | Investigate |
| **> 5000 ppm sustained** | ~10 % LEL | **Safety alarm** | Evacuate/ventilate, notify |

**MQ‑4 → CH₄ — sewer gas / safety channel**

Methane LEL = 5 % = **50 000 ppm**. MQ‑4's 200–10 000 ppm range therefore covers **0.4 % – 20 % LEL**,
which is exactly the useful early‑warning window.

| Band | CH₄ | % LEL | Action |
|---|---|---|---|
| Ambient | ~2 ppm (below range) | — | Green |
| Elevated | 200 – 1000 ppm | 0.4–2 % | Sewer gas ingress / dry trap — check plumbing |
| Warning | **1000 – 5000 ppm** | 2–10 % | Amber, ventilate |
| **Alarm** | **> 5000 ppm** | **> 10 % LEL** | **Red — standard industrial gas‑alarm trigger** |
| Evacuate | > 10 000 ppm | > 20 % LEL | Evacuate |

**MQ‑9 → CO — only valid with heater cycling (§3.3)**

| Band | CO | Basis | Action |
|---|---|---|---|
| Normal | < 9 ppm | EPA 8‑h IAQ guidance | Green |
| Elevated | 9 – 35 ppm | NIOSH REL TWA = 35 ppm | Amber, ventilate |
| Alarm | **35 – 50 ppm** | NIOSH REL / OSHA PEL 50 ppm | Red |
| Danger | > 200 ppm | NIOSH ceiling 200 ppm; symptoms appear | Evacuate |
| — | 1200 ppm | IDLH | — |

For context, UL 2034 domestic CO alarms must **not** alarm below 30 ppm for 30 days, and must alarm at
70 ppm within 1–4 h, 150 ppm within 10–50 min, 400 ppm within 4–15 min. If you want the TMS to behave
like a recognised CO alarm, mirror that time‑weighted behaviour rather than a bare threshold.

**Realistically:** in a toilet block with no combustion source, CO should be flat at baseline forever. If
MQ‑9 reads meaningful CO, the likely causes are a nearby diesel generator (very plausible at a Kumbh site),
a gas geyser, or vehicle exhaust drawn in through a vent. That is worth detecting.

---

## 6. The maths: ADC → ppm

### 6.1 Chain

```
                    R_s (sensor)        R_L (load)
    +5V ──────────/\/\/\──────┬────────/\/\/\──────── GND
                              │
                              ├──> V_RL ──[ divider ]──> ESP32-S3 ADC (max ~3.1 V)
```

1. **ADC → volts.** ESP32‑S3 ADC1 (GPIO1–GPIO10) only; ADC2 is unusable while Wi‑Fi is active. 12‑bit,
   12 dB attenuation gives roughly 0.15–3.1 V usable and is **noticeably non‑linear at both ends** — use
   the ESP‑IDF calibration API (`esp_adc_cal` / `adc_cali_*`) rather than a bare `raw * 3.3 / 4095`.
   Oversample: 32–64 samples, discard outliers, take the median.

2. **Undo the divider.** `V_RL = V_adc × (R_top + R_bot) / R_bot`.
   ⚠️ **I do not know your divider values** — the shield shows ~3 resistors + 1 capacitor per sensor
   (R6/R7/R8+C15 for MQ‑4, R9/R10/R11+C16 for MQ‑2, R13/R14/R20+C17 for MQ‑137, R15/R16/R17/R22+C18 for
   MQ‑9). I need the schematic (§10).

3. **Volts → sensor resistance**, straight from the datasheets:
   ```
   R_s = (V_C / V_RL − 1) × R_L
   ```

4. **Normalise:** `ratio = R_s / R₀`, where R₀ is the per‑unit calibration constant from §7.

5. **Ratio → ppm:** `ppm = a × ratio^b` (power law; the datasheet curves are log‑log straight lines).

### 6.2 Curve coefficients — with a warning

Derived from the commonly used MQ sensor libraries (the datasheets publish these curves only as images):

| Sensor | Clean‑air Rs/R₀ | Gas | a | b |
|---|---|---|---|---|
| **MQ‑2** | 9.83 | LPG | 574.25 | −2.222 |
| | | Propane | 658.71 | −2.168 |
| | | H₂ | 987.99 | −2.162 |
| | | Alcohol | 3616.1 | −2.675 |
| | | CO | 36974 | −3.109 |
| **MQ‑4** | 4.4 | **CH₄** | **1012.7** | **−2.786** |
| | | LPG | 3811.9 | −3.113 |
| | | Smoke | 3.0e7 | −8.308 |
| **MQ‑9** | 9.6 | LPG | 1000.5 | −2.186 |
| | | **CO** | **599.65** | **−2.244** |
| | | CH₄ | 4269.6 | −2.648 |
| **MQ‑137** | see warning | NH₃ | see warning | see warning |

**⚠️ MQ‑137 coefficients are not trustworthy, and here is the proof.**
The two most commonly circulated MQ‑137 fits both use a log‑log slope of about **−0.264**
(`ppm = 10^((log10(ratio) − b) / m)` with `m ≈ −0.263`). A slope of −0.264 implies the resistance ratio
changes by `2^−0.264 = 0.83` per doubling of concentration. But the **datasheet specifies α ≤ 0.6**
(R₁₀₀ppm/R₅₀ppm), which requires a slope of **at least −0.737** in magnitude. The published fits are off
by roughly 3× in slope, and the two sources disagree with each other on the intercept
(one uses +0.42, another −0.241 — those give clean‑air answers of 39 ppm vs 0.12 ppm from the same input).

**Do not ship MQ‑137 ppm from a library constant.** Either calibrate against a reference (§7.3) or —
better for this product — report MQ‑137 as a **calibrated odour index** rather than a physical ppm, and
be honest in the UI about which numbers are physical and which are indices.

### 6.3 Temperature and humidity compensation

All four datasheets publish a temperature/humidity dependence curve, and the effect is large: MQ‑2's
Rs/R₀ spans roughly **0.5 to 1.9** across −20 °C to +50 °C and 30–85 % RH. That is a ±2× swing on the raw
ratio — **bigger than the odour signal you are trying to detect.** This is why the BOM has a temperature &
humidity sensor, and using it is not optional.

The widely used first‑order correction (originally fitted for MQ‑135, reference point 20 °C / 33 % RH):

```
cFactor = CORA·t² − CORB·t + CORC − (h − 33)·CORD
    CORA = 0.00035,  CORB = 0.02718,  CORC = 1.39538,  CORD = 0.0018
    t = temperature °C,  h = relative humidity %
R_s_corrected = R_s / cFactor
```

Caveats: this was fitted for MQ‑135, not for MQ‑137/2/4/9, and it is linear in humidity where the real
curves are not. For this product I recommend:

1. Apply the correction above as a first pass — it is much better than nothing.
2. **Also record raw R_s, temperature and humidity in every telemetry record.** With a fleet of units you
   can fit a proper per‑model correction from real data after a few weeks, and re‑derive offline without
   touching firmware. Do not throw away the raw values.
3. Reject readings taken during condensing conditions (RH > 90 % and falling temperature) — flag them
   rather than publishing them.

---

## 7. Calibration and R₀

### 7.1 Burn‑in (one time, at manufacture)

- **MQ‑2, MQ‑9, MQ‑137: over 48 hours. MQ‑4: over 24 hours.** These are datasheet numbers, not folklore.
  A sensor read at 30 minutes will drift for days afterwards.
- Do this in the factory/office in clean air, powered continuously, before the unit ships.
- Sensors stored a long time without power drift reversibly and need a **longer** aging time — relevant if
  boards were assembled well before deployment.

### 7.2 R₀ capture (per unit, after burn‑in)

```
R_s_air = (V_C / V_RL − 1) × R_L          // measured in clean air
R₀      = R_s_air / cleanAirRatio          // cleanAirRatio per §6.2
```

Store R₀ **per sensor, per unit, in NVS** along with the temperature and humidity at capture time. R₀
varies several‑fold between individual sensors of the same part number — a shared constant across the
fleet will produce nonsense. Refuse to publish ppm until R₀ exists.

### 7.3 If you want real MQ‑137 accuracy

The only honest way is a two‑point calibration against a known NH₃ concentration. Options in rough order
of cost:

1. **Reference instrument:** co‑locate a calibrated electrochemical NH₃ meter for a week on one unit,
   regress your R_s/R₀ against it, and use that fit fleet‑wide. This is the pragmatic choice.
2. **Ammonium hydroxide headspace:** a sealed chamber with a known dilution gives a repeatable, if not
   traceable, mid‑range point. Good enough to fix the slope.
3. **Certified span gas** (50 ppm NH₃ in N₂) + a flow chamber. Traceable, but a real lab exercise.

Given the deployment (mass gathering, public health visibility), option 1 is worth the money for at least
one unit — it converts the entire fleet's output from "an index" to "a number you can defend".

### 7.4 Drift management in the field

SnO₂ sensors drift over months, and yours are in a corrosive environment (§3.5). Plan for it:

- **Rolling baseline:** maintain a long‑window (e.g. 7‑day) low percentile of corrected R_s per sensor as
  the "current clean" reference. Odour index is computed against this, not against factory R₀.
- **Re‑baseline on a verified cleaning event:** when RFID says a cleaner attended, and the gas signals then
  settle for N minutes, take that settled value as the new clean reference. This is the highest‑quality
  baseline you will get, and it is free.
- **Guard rails:** if the rolling baseline moves more than a set factor (e.g. 3×) from factory R₀, flag the
  sensor as needing service rather than silently tracking it. That is how you catch a poisoned sensor.
- **Replacement schedule:** budget for gas sensor replacement — the datasheets explicitly warn that H₂S and
  chlorine cause "sincere sensitivity attenuation", and both are present in a public toilet. Make the
  sensors socketed (they appear to be) and treat them as a consumable.

---

## 8. Recommended algorithm: Toilet Cleanliness / Odour Index

Given §1 point 3 (real signal sits below MQ‑137's specified floor), do **not** build the product logic on
absolute ppm. Build it on normalised deviation from the learned baseline, and publish ppm alongside for
the ones that are physically meaningful.

**Per sensor, per sample:**

```
r      = R_s_corrected / R_baseline        // R_baseline from §7.4, not factory R₀
dev    = max(0, −log10(r))                 // 0 when clean; grows as resistance falls
index  = clamp(100 × dev / dev_full, 0, 100)
```

`dev_full` is the deviation corresponding to the top of each sensor's meaningful band — calibrate it from
the §5.2 red thresholds (e.g. for MQ‑137, the deviation observed at ~25 ppm NH₃).

**Combined Odour Index (OI):**

```
OI = clamp( 0.70 × I_MQ137          // ammonia — the dominant toilet odorant
          + 0.20 × I_MQ4            // faecal/sewer corroboration
          + 0.10 × I_MQ2            // generic VOC
          , 0, 100 )
```

MQ‑9 stays **out** of the odour index — it is a safety channel, not a cleanliness channel (and may not be
valid at all, per §3.3).

**Suggested state machine:**

| OI | State | Action |
|---|---|---|
| 0–25 | **Clean** | Indicator green |
| 25–50 | **Acceptable** | Green |
| 50–70 | **Attention** | Amber; queue cleaning at next round |
| 70–85 | **Dirty** | Amber/red; dispatch cleaner, trigger auto‑flush/spray via relay |
| 85–100 | **Foul** | Red; dispatch immediately, force extract fan |

**Overrides that must bypass the index entirely:**

- MQ‑4 > 10 % LEL, or MQ‑2 sustained high → **safety alarm**, independent path, no averaging delay.
- Any sensor open/short/out‑of‑range → **fault state**, not "clean". A dead sensor must never read green.
- Cleaning hold‑off after an RFID cleaner scan → suppress odour alarms for a configured window.

**Debounce everything.** Require N consecutive samples above a threshold before changing state, and use
hysteresis on the way down. A single ADC sample must never dispatch a cleaner or fire a pump.

**Cross‑check with usage count.** You have a person‑count sensor and a door/reed switch. Odour that rises
with footfall is normal; odour rising with *zero* footfall means a blockage, a dry trap, or a spill — a
genuinely different fault, and detectable only by combining the two.

---

## 9. Hardware findings and risks

These came out of the research and are worth resolving before firmware is finalised.

**① Power budget looks tight — please verify the supply rating.**
Heater current at 5 V, from the datasheets:

| Sensor | P_H | Current @ 5 V |
|---|---|---|
| MQ‑2 | ≤ 900 mW | ≤ 180 mA |
| MQ‑137 | ≤ 900 mW | ≤ 180 mA |
| MQ‑4 | < 750 mW | ≈ 150 mA |
| MQ‑9 | < 340 mW | ≈ 68 mA |
| **Heaters total** | **≈ 2.9 W** | **≈ 578 mA** |

Add ESP32‑S3 with Wi‑Fi (≈150 mA average, **500 mA peak** during TX), MFRC522 RFID, the indicator, and the
relay card coil current. **Continuous 5 V demand is comfortably over 800 mA, with peaks past 1.2 A.**

The BOM lists "Hylink Power Supply" at ₹205. If that is an **HLK‑PM01, it is a 3 W / 5 V / 600 mA module —
undersized before the ESP32 even transmits.** Symptoms of running it marginal are nasty and hard to debug:
the 5 V rail sags, which changes V_C, which changes every gas reading, which looks like a drifting sensor.
Recommend **HLK‑10M05 (10 W) or HLK‑PM12‑class (12 W)**, plus bulk capacitance on the 5 V rail. Please
confirm the exact part.

**② Confirm the MQ‑9 heater drive** (§3.3). This determines whether you have a CO channel at all.

**③ I need the ADC divider values** to compute R_s at all (§6.1 step 2).

**④ H₂S gap** (§1 point 2). If a revision is possible, adding an **MQ‑136** would cover the odorant humans
are most sensitive to, and would substantially improve the cleanliness index. If not possible for V1.2,
document that H₂S is inferred, not measured.

**⑤ No silicone anywhere near the sensors** — permanent poisoning (§3.5). Check enclosure gasket and any
potting/conformal coating.

**⑥ Condensation and washdown.** Toilet blocks get hosed down. The datasheets are explicit that water on
the element reduces sensitivity permanently. Confirm the enclosure's ingress rating and that the sensor
vents cannot take a direct jet.

**⑦ ADC channel assignment.** ESP32‑S3 ADC1 is GPIO1–GPIO10 only, and ADC2 cannot be used with Wi‑Fi on.
The board silkscreen exposes IO1, IO2, IO4, IO5, IO6, IO7 — consistent with four gas channels plus spares.
Confirm from the schematic.

**⑧ Self‑heating inside a sealed box.** Nearly 3 W of heaters in a small enclosure will raise the internal
temperature well above ambient — which, per §6.3, directly biases every reading. Mount the temperature/
humidity sensor **in the sampled airflow near the gas sensors**, not next to the ESP32 or the PSU, or the
compensation will make things worse rather than better.

---

## 10. What I need to write the firmware

The research is done; these are the blockers for turning it into code.

1. **Schematic / netlist for TMS‑V1.2** (the file you sent was the cost analysis, not the schematic).
   Specifically: which ESP32‑S3 GPIO each MQ analog output lands on; the R_L value per sensor; the divider
   resistor values; and whether any heater is GPIO‑switchable.
2. **Exact temperature/humidity part** — DHT11, DHT22, AHT20, SHT30/31? (Affects driver + accuracy of §6.3.)
3. **MQ‑9 heater drive** — switchable or hard‑tied 5 V?
4. **Exact Hylink PSU part number** (§9①).
5. **RFID reader part** — MFRC522 on the SPI header (RST/GND/MISO/MOSI/SCK/SS)?
6. **Person‑count sensor and "Ride Switch"** — I read "Ride Switch" as a **reed switch** for door
   open/close; please confirm. And is the person counter the IR header, the PIR/motion header, or both?
7. **Firmware target** — Arduino‑ESP32 or ESP‑IDF? (I would recommend ESP‑IDF for the ADC calibration API
   and OTA robustness on a large deployment, but Arduino is faster to iterate.)
8. **Backend/telemetry** — MQTT? HTTP? What server, what payload schema, what offline‑buffering
   expectation? At a Kumbh site, connectivity will be intermittent — units need to buffer to flash.

Answer 1–3 and I can write the gas subsystem immediately; the rest can follow.

---

## 11. Sources

Primary datasheets (downloaded and text‑extracted):

- [MQ-2 — Semiconductor Sensor for Combustible Gas](https://github.com/AcoptexCom/arduino/blob/master/MQ2%20GasSmoke%20Sensor/Datasheet/MQ2%20Gas%20sensor.pdf)
- [MQ-4 — Hanwei Electronics Technical Data](https://github.com/CytronTechnologies/Gas_Sensor/blob/master/Natural%20Gas%20Sensor%20(SN-MQ4)/usr_attachment-MQ-4%20datasheet.pdf)
- [MQ-9 — Hanwei Electronics Technical Data](https://raw.githubusercontent.com/SeeedDocument/Grove-Gas_Sensor-MQ9/master/res/MQ-9.pdf)
- [MQ-137 — Semiconductor Sensor for Ammonia](https://github.com/Microbitz/MQ-137)

Curve coefficients and correction factor:

- [miguel5612/MQSensorsLib](https://github.com/miguel5612/MQSensorsLib) — per-gas a/b coefficients, clean-air ratios, calibrate()/readSensor() implementation
- [SolderedElectronics/Soldered-MQ-Gas-Sensor-Arduino-Library](https://github.com/SolderedElectronics/Soldered-MQ-Gas-Sensor-Arduino-Library) — `sensorConfigData.h` per-sensor table
- [carry0987/MQ-137](https://github.com/carry0987/MQ-137/blob/master/mq-137.ino) — MQ-137 slope/intercept (see §6.2 warning)

Exposure limits and restroom measurements:

- [NIOSH — Hydrogen sulfide IDLH](https://www.cdc.gov/niosh/idlh/7783064.html) and [1988 OSHA PEL Project: Hydrogen Sulfide](https://www.cdc.gov/niosh/chemicals/pel88/pell-pages/7783-06.html)
- [NIOSH — Ammonia IDLH](https://www.cdc.gov/niosh/idlh/7664417.html)
- [OSHA Annotated PELs, Table Z-1](https://www.osha.gov/annotated-pels/table-z-1)
- [Inhibition of ammonia and hydrogen sulphide as faecal sludge odour control in dry sanitation toilet facilities (Scientific Reports)](https://www.nature.com/articles/s41598-021-97016-w) — measured 0.22/3.17 ppm H₂S, 3.16/6.88 ppm NH₃
- [CFD studies on the spread of ammonia and hydrogen sulfide pollutants in a public toilet (ScienceDirect)](https://www.sciencedirect.com/science/article/abs/pii/S2352710221015862)
- [EPA — Carbon Monoxide's Impact on Indoor Air Quality](https://www.epa.gov/indoor-air-quality-iaq/carbon-monoxides-impact-indoor-air-quality)
- [CO2Meter — Carbon monoxide levels chart / UL 2034 alarm thresholds](https://www.co2meter.com/blogs/news/carbon-monoxide-levels-chart)

ESP32-S3 ADC:

- [ESP-IDF Programming Guide — ADC (ESP32-S3)](https://docs.espressif.com/projects/esp-idf/en/v4.4.6/esp32s3/api-reference/peripherals/adc.html)

Temperature/humidity dependence:

- [Davide Gironi — MQ gas sensor correlation function against temperature and humidity](https://davidegironi.blogspot.com/2017/07/mq-gas-sensor-correlation-function.html)
- [One Transistor — Influence of temperature and humidity on MQ gas sensors](https://www.onetransistor.eu/2023/01/mq-sensors-temperature-humidity.html)
