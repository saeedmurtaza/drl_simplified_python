# drl_simplified_python

Simplified ("proactive-control") successor to the DQN controller in [Full-DRL-Heating-codes](https://github.com/saeedmurtaza/Full-DRL-Heating-codes). Instead of a neural-network agent, this repository uses:

* a **grey-box 2R2C thermal model** of the building, identified from field data (`scipy.optimize.curve_fit`),
* a **thermodynamic energy model** of the VRF outdoor unit (R410A cycle via NIST REFPROP + linear regression on compressor work and operating rate),
* a **two-coefficient linear policy** for the ODU target pressure, whose coefficients are chosen weekly by simulating a small scenario grid against a comfort + energy reward,
* a **10-minute dispatcher** that applies the selected policy to live data and writes the target pressure to SQLite.

The cooling (summer) side is implemented in Python notebooks. The heating (winter) side exists only as an analysis workbook (power/evaporating-temperature regressions and a defrost study) and has no Python code yet.

Both ODUs serve one office building: **ODU#1 = 3rd floor, 8 HP, 4 indoor units; ODU#2 = 2nd floor, 10 HP, 5 indoor units.** Pressures are gauge values in kgf/cm² as logged by the units.

---

## Table of contents

1. [Repository layout](#1-repository-layout)
2. [Installation](#2-installation)
3. [External data and files not in the repo](#3-external-data-and-files-not-in-the-repo)
4. [Summer (cooling) pipeline](#4-summer-cooling-pipeline)
   - 4.1 [Thermal model identification — `2R2C_ODU*_2W5P_0817.ipynb`](#41-thermal-model-identification--2r2c_odu_2w5p_0817ipynb)
   - 4.2 [Weekly policy selection — `select_policy*.ipynb`](#42-weekly-policy-selection--select_policyipynb)
   - 4.3 [10-minute dispatch — `make_order*.ipynb`](#43-10-minute-dispatch--make_orderipynb)
   - 4.4 [Design document — `ProactiveControl간단버젼(230912).xlsm`](#44-design-document--proactivecontrol간단버젼230912xlsm)
5. [Winter (heating) workbook](#5-winter-heating-workbook)
6. [Database tables](#6-database-tables)
7. [Known issues and things to fix before reuse](#7-known-issues-and-things-to-fix-before-reuse)
8. [Running the summer loop](#8-running-the-summer-loop)

---

## 1. Repository layout

```
Simplified DRL Python Codes/
├── Summer(Cooling mode)/
│   ├── 2R2C_ODU1_2W5P_0817.ipynb          # fit 2R2C building + HVAC-gain model, ODU#1
│   ├── 2R2C_ODU2_2W5P_0817.ipynb          # same for ODU#2 (lower half still holds ODU#1 outputs)
│   ├── select_policy.ipynb                # weekly: simulate 18 policy scenarios, pick best → 1st_policy.csv
│   ├── select_policy_odu2.ipynb           # same for ODU#2 → 2nd_policy.csv
│   ├── make_order.ipynb                   # every 10 min: live target pressure → odu1_drl_policy
│   ├── make_order_odu2.ipynb              # same for ODU#2 → odu2_drl_policy
│   └── ProactiveControl간단버젼(230912).xlsm   # Korean design doc: model equations, parameter bounds,
│                                           #   identified parameters, policy/reward definition
└── Winter(Heating mode)/
    └── 202311071328(OUD1_power)_231116.xlsm   # 73 130 rows of 10-s ODU#1 heating data + REFPROP
                                               #   formulas; evaporating-temp & power regressions; defrost study
```

`_2W5P` = "2-week windows, 5 periods" (an earlier ensemble approach that survives in the lower half of the notebooks). `0817` / `230912` / `231116` are YYMMDD edit dates.

---

## 2. Installation

Python 3.x with:

```bash
pip install "pandas<2.0" numpy scipy scikit-learn matplotlib openpyxl jupyter
pip install ctREFPROP          # NIST REFPROP Python wrapper
```

* `pandas<2.0` is required: `select_policy*.ipynb` uses `DataFrame.append`, removed in pandas 2.0.
* **NIST REFPROP** (licensed, Windows DLL) must be installed at `C:/Refprop_dll` with the `R410A.MIX` mixture file; the notebooks call `REFPROPFunctionLibrary('C:/Refprop_dll')`. Without it `select_policy*` cannot compute cycle enthalpies. The 2R2C and `make_order` notebooks import it but do not use it.
* The `.xlsm` workbooks embed the **REFPROP Excel add-in (REFPROP.XLA)** VBA and expect `C:\Refprop_dll\REFPROP.DLL`; the winter workbook also uses custom VBA functions (`Ts_410A`, `상대습도`, `이슬점`, `포화압력`, `ASHREA_HumidityRatio_ARD`). Open with macros enabled on a machine with REFPROP, otherwise formula cells show cached values only.
* `sqlite3` is from the standard library.

---

## 3. External data and files not in the repo

| Path (as hard-coded) | Used by | Contents |
|---|---|---|
| `ODU#1_5min_v2_weekend_avg.xlsx`, `ODU#2_5min_v2_weekend_avg.xlsx` | 2R2C notebooks | 5-min averaged BMS data, 2022-06-06 → 2022-07-21, ≈12 860 rows; columns `timestamp, hour, Weekday, IAT, IDUk_Operation_On&OFF, IDUk_Setting_temperature, IDUk_Current_Indoor_fan_wind, IDUk_Indoor_temperature, IDUk_Indoor_unit_ABS_Req_capacity, IDUk_Thermo_status, ODU1_Outdoor_temperature_(1_second_average), ODU1_target_pressure, ODU1_Low_pressure_(1_second_value), ODU1_operation_mode, …` |
| `C:/KHU_DRL/NED_SQLite.db` | `select_policy*`, `make_order*` | site database, same schema as the Full-DRL repo (§6) |
| `C:/KHU_DRL/1st_policy.csv`, `2nd_policy.csv` | written by `select_policy*`, read by `make_order*` | one row, no header: `coef_amb, coef_delta, offset` |
| `C:/KHU_DRL/1st_score_<YYYYmmdd_HHMM>.csv`, `2nd_score_…` | `select_policy*` | 18-row scenario score table |
| `C:/KHU_DRL/output/ODU{1,2}_<ts>_<ii>_<jj>_<kk>.csv` | `select_policy*` | 5-day simulated trajectory per scenario |
| `C:/KHU_DRL/ODU{1,2}_policy_history.txt` | `select_policy*` | append-only log `ts / coef_amb / coef_delta / offset` |

---

## 4. Summer (cooling) pipeline

```
 field data (5-min xlsx)                live SQLite (1-min)
        │                                     │
        ▼                                     │
 2R2C_ODUn.ipynb  ──► Ri,Ro,Ci,Ce,Atp,Aac,Across,Aih,C_0,C_1  (pasted by hand into select_policy)
                                              │
                      weekly ◄────────────────┘ last 3 days OAT
                 select_policy_*.ipynb : 18 scenarios × 5-day simulation → best (coef_amb, coef_delta, offset)
                                              │   C:/KHU_DRL/{1st,2nd}_policy.csv
                      every 10 min ◄──────────┘ last 10 min of oduN_data
                 make_order_*.ipynb : target LP = f(policy, OAT, IAT, ST) + calibration → oduN_drl_policy
```

### 4.1 Thermal model identification — `2R2C_ODU*_2W5P_0817.ipynb`

**Model** (explicit Euler, `dt = 300 s`; `Ti` indoor, `Te` wall, `To` outdoor):

```
Te[t] = Te[t-1] + dt/Ce · ( (Ti[t-1]-Te[t-1])/Ri + (To[t-1]-Te[t-1])/Ro )

if op_rate[t-1] == 0 (unit idle):
    Ti[t] = Ti[t-1] + dt/Ci · ( (Te[t-1]-Ti[t-1])/Ri + Aih·inner_heat[t-1] + C_1 )
else:
    Ti[t] = Ti[t-1] + dt/Ci · ( (Te[t-1]-Ti[t-1])/Ri
                               + Atp·P[t-1] + Aac·op_rate[t-1] + Across·P[t-1]·op_rate[t-1]
                               + Aih·inner_heat[t-1] + C_0 )
```

Units as documented in the design workbook: `Ri, Ro` in K/kW, `Ci, Ce` in kJ/K.

**Inputs built from the 5-min data**

| Symbol | Column / definition |
|---|---|
| `To` | `ODU1_Outdoor_temperature_(1_second_average)` |
| `P` | `ODU1_Low_pressure_(1_second_value)` (ODU low pressure), set to 0 while `op_rate == 0` |
| `op_rate` | Σₖ `IDUk_Current_Indoor_fan_wind · IDUk_Indoor_unit_ABS_Req_capacity` ÷ (0.8·250·10) for ODU#1 (8 HP), ÷ (1.0·250·10) for ODU#2 (10 HP) |
| `inner_heat` | 1 on Mon–Fri when `8 < hour < 18`, else 0 |
| `Ti` (target) | `IAT` (capacity-weighted indoor temperature) |

**Two-stage fit**

1. *Envelope* (`Ri, Ro, Ci, Ce, te_0`) on an idle window with no HVAC — Fri 2022-07-01 22:00 → Mon 07-04 07:00 — using only `To`; bounds `Ri [0.01, 1]`, `Ro [0.01, 5]`, `Ci [1000, 2000]`, `Ce [2e4, 2.5e5]`, `te_0 [20, 45]` (derived in the `(변수의범위)` sheet from a 211 m² floor, 4 m height, 200 mm concrete walls, h = 10–200 W/m²K).
2. *HVAC and internal-gain terms* (`Atp, Aac, Across, Aih, te_0, C_0, C_1`) on 2022-07-01 05:00 → 07-22 23:59 with the envelope fixed; bounds `[0, -35, -28, 0, 15, -5, -5]` to `[5, -3, 28, 10, 40, 5, 5]`.

**Identified values (notebook outputs; also tabulated in the design workbook)**

| | ODU#1 (3F, 8 HP) | ODU#2 (2F, 10 HP) |
|---|---|---|
| `Ri` | 0.296298 | 0.172560 |
| `Ro` | 1.262322 | 1.798633 |
| `Ci` | 1263.760 | 1157.592 |
| `Ce` | 81085.68 | 100446.99 |
| envelope R² | 0.443 | 0.356 |
| `Atp` (b) | 0.216114 | 0.087698 |
| `Aac` (a) | −28.649686 | −35.0 (hit lower bound) |
| `Across` (c) | 2.307840 | 2.642250 |
| `Aih` | 0.390904 | ≈ 0 |
| `te_0` | 26.228 | 26.361 |
| `C_0` (d) | −0.585024 | 2.227854 |
| `C_1` (err) | 2.251970 | 2.023524 |
| HVAC-stage R² | 0.274 | 0.244 |

Everything below the markdown cell `# 여기서부터는 Test용 코드임` ("test code from here") is the earlier 2W5P ensemble experiment (five overlapping two-week windows, average prediction) and is not used downstream. In `2R2C_ODU2…` those cells still show ODU#1's cached outputs.

### 4.2 Weekly policy selection — `select_policy*.ipynb`

**Policy** (identical in `select_policy*` and `make_order*`):

```python
target_LP = coef_amb*(OAT - 30) + coef_delta*(IAT - ST) + offset + 8     # kgf/cm²G low pressure
target_LP = clip(target_LP, 6.5, 9.5); 0 if unit off; rounded to 0.1
```

**Scenario grid** (3 × 3 × 2 = 18): `coef_amb ∈ {-0.3, -0.1, 0.1}`, `coef_delta ∈ {-0.3, -0.15, 0.0}`, `offset ∈ {0, 0.2}`.

**Simulation per scenario** (5 days × 288 five-minute steps):

* Outdoor profile: last 4320 rows (3 days) of `oduN_data.ODU_outtemp`, averaged to a 5-min diurnal profile.
* `set_temp = 24 °C`, `Ti[0] = 25 °C`, `Te[0] = mean(OAT profile)`.
* Unit on 08:30–17:59; forced off when `Ti ≤ set_temp − 1`; when on, `op_rate = 0.75`. Internal heat 09:00–17:59 (also used as the occupancy flag).
* `Ti`, `Te` advanced with the §4.1 equations and the identified parameters (hand-pasted into a cell; `odu1_ti_t` / `odu1_te_t`).
* **Energy** for each running step:
  1. `ΔT_cond = (9 − LP)·1.5 + op_rate·5 + (Ti − Tsat(LP) − 12)·0.5` (engineering estimate; not validated — see design doc)
  2. `P_cond = Psat_R410A(OAT + ΔT_cond)` in gauge kgf/cm², floored at 25
  3. `w_comp = h(P_cond, s₁) − h₁`, with `h₁, s₁` at suction `LP` and `Tsat(LP) + 5 K` superheat (isentropic compression, REFPROP)
  4. `power = −3220 + 3305.5·op_rate + 138.72·w_comp` (ODU#1) / `−2420 + 5795·op_rate + 109.7·w_comp` (ODU#2) [W]; ×1.5 on the first step after a stop
* **Reward**
  * comfort = mean over occupied steps of `100 if Ti−Tset<1, 90 if <2, 70 if <3, 50 if <4, else 0`
  * energy = `−100 · E_kWh / E_std`, `E_std = 5 d · 8 h · 8 HP · 2.9 kW / 3.0 · 0.75 = 232 kWh`
  * total = comfort + energy

Outputs: per-scenario trajectory CSVs, the score table, the best `(coef_amb, coef_delta, offset)` to `1st_policy.csv` (`2nd_policy.csv` for ODU#2), and a line in `ODUn_policy_history.txt`. The design document specifies running this **once a week**.

### 4.3 10-minute dispatch — `make_order*.ipynb`

1. Read the policy coefficients from `C:/KHU_DRL/{1st,2nd}_policy.csv`.
2. `SELECT * FROM oduN_data ORDER BY timestamp DESC LIMIT 10` (last 10 minutes). Encode `운전 On/Off` → 1/0, `일반 운전/정지` → 1/0.
3. Capacity-weighted `ST` and `IAT` over running IDUs; if no IDU is running, `IAT` = capacity-weighted over all IDUs and `ST := IAT`. Average the 10 rows.
4. `target_LP = policy(OAT_avg, ST_avg, IAT_avg, op_status=1)`.
5. Calibration: over the 10 rows and all IDUs, count samples where the IDU is on (`i_operation`) and where it is on **and** `intemp − settemp < 1` (`i_cali`); add `round(i_cali / i_operation · 0.5, 1)` — up to +0.5 kgf/cm² when every running IDU is already satisfied (raises LP → lower capacity).
6. `INSERT INTO oduN_drl_policy (timestamp, ODU_targetpress, ODU_onoff) VALUES (now, target_LP, '정상')`.

The ODU#2 notebook keeps the variable name `odu1_data` but queries `odu2_data`, includes `IDU4`, and writes to `odu2_drl_policy`.

### 4.4 Design document — `ProactiveControl간단버젼(230912).xlsm`

Three sheets (Korean):

* **온도예측모델** — 2R2C model definition, identification method (envelope from idle weekend, `f2 = a·op_rate + b·LP + c·op_rate·LP + d` for the unit), the parameter tables above, and the policy-selection/reward scheme of §4.2 (12-scenario grid before 2023-09-12, 18 after adding the offset).
* **(변수의범위)** — physical derivation of the `curve_fit` bounds.
* **에너지예측모델** — energy-model derivation. Data filter: all IDU on/off and thermo states fixed ≥ 6 min, ODU in `일반 운전` ≥ 6 min, 5-min averages, drop |P − target| ≥ 1 kgf/cm² and compressor frequency ≤ 20 Hz. Regression summaries: ODU#1 R² 0.8887 (S 332.5), ODU#2 R² 0.8632 (S 370.8). Notes that the high-pressure ΔT model could not be fitted from data (almost all 2022 data was controlled at 25 kgf/cm²G) and is an unvalidated engineering estimate.

---

## 5. Winter (heating) workbook

`Winter(Heating mode)/202311071328(OUD1_power)_231116.xlsm` (19.6 MB) — ODU#1 heating-mode analysis, no Python counterpart.

**Sheet `202311071328`** — 73 130 rows at 10-s resolution, 2022-12-15 16:50 → 2023-02-07 18:23:

| Column | Content |
|---|---|
| `timestamp, od_temp, OTW, avg_id_temp, cond_temp, eva_temp, cond_press, op_rate, power` | logged values (OTW = outdoor wet-bulb) |
| `고압포화온도` | `=Ts_410A(cond_press)` saturation temp at high pressure |
| `고압측delta_t` | `= 고압포화온도 − avg_id_temp` |
| `예측 저압 포화온도` | `= 9.322 + 1.00707·OTW + 0.29483·avg_id_temp − 0.30791·cond_temp − 8.0214·op_rate` |
| `저압` | `=pressure("R410A.mix","T","MKS", Tsat_pred + 273.15)` |
| `h1, s1` | suction enthalpy/entropy at `(저압, Tsat_pred + 5 K)` |
| `h2` | `=Enthalpy("R410A.mix","PS","MKS",(cond_press+1.033)·98.066, s1)` |
| `delta_h` | `h2 − h1` (compressor work, kJ/kg) |
| unnamed col S | `= 1677 − 9306.6·op_rate + 21.86·delta_h + 304.04·op_rate·delta_h` — **heating power model, R² 0.878** |
| `상대습도, delta_e, delta_wb` | RH from dry/wet bulb; `od_temp − eva_temp`; `OTW − eva_temp` |

**Sheet `Sheet1`** — Minitab-style regression summaries for the evaporating temperature (three variants: dry-bulb + wet-bulb R² 0.933; wet-bulb only R² 0.859, *adopted as physically most consistent*; dry-bulb only R² 0.932) and the power model above.

**Sheet `제상` (defrost)** — frost-onset study: dew point vs. evaporating temperature per outdoor temperature (7 → −23 °C, RH 0.85), ΔT growth factor ≈ 1.04 per minute, time-to-frost; assumptions for evaluation: defrost lasts 15 min and consumes 25 % of full-load power (op_rate 100 %, target 30 kgf/cm²); proposed rule — if predicted time to defrost < 30 min, revert to normal control.

---

## 6. Database tables

Same SQLite file and schema as the Full-DRL repository (`NED_SQLite.db`), here expected at `C:/KHU_DRL/`:

| Table | Role here |
|---|---|
| `odu1_data`, `odu2_data` | 1-min BMS log: `timestamp, IDUk_onoff (운전 On/Off), IDUk_settemp, IDUk_intemp, IDUk_capa, ODU_outtemp, ODU_targetpress, ODU_highpress, ODU_power, ODU_onoff (일반 운전/정지/…)`; k = 0..3 (ODU#1), 0..4 (ODU#2) |
| `odu1_drl_policy`, `odu2_drl_policy` | written by `make_order*`: `timestamp, ODU_targetpress, ODU_onoff` (`ODU_onoff` is the literal `'정상'` here) |

---

## 7. Known issues and things to fix before reuse

1. **Hard-coded Windows paths** everywhere: `C:/KHU_DRL/…`, `C:/Refprop_dll`, and the data `.xlsx` names. Move to a config/env variable.
2. **Gauge→absolute pressure bug** in `select_policy*.ipynb::condensing_temp_delta_t`: `(target_press*1.03)*98` multiplies by 1.03 where the other two REFPROP helpers add it (`(p + 1.03)*98`). The evaporating temperature used for the condensing-ΔT estimate is therefore slightly wrong.
3. `E_std` in the energy score uses **8 HP for both units**; ODU#2 is 10 HP, so its energy score is penalised ≈ 25 % harder.
4. Identified parameters are **pasted by hand** from the 2R2C notebooks into `select_policy*`; there is no file handoff. If you re-run identification, update the constants cell.
5. `2R2C_ODU2_2W5P_0817.ipynb` lower half contains stale ODU#1 outputs (copied notebook, cells not re-executed).
6. `make_order*` writes `ODU_onoff = '정상'` (string) — the Full-DRL deployment writes `일반 운전`/`정지` for ODU#1 and 1/0 for ODU#2. Consumers of `oduN_drl_policy` must accept all three encodings or the writers should be aligned.
7. `select_policy*` uses `DataFrame.append` (pandas < 2.0 only) and inserts into `df_result` every step (slow; 18 × 1440 appends). The 5-min OAT profile indexes `df_period.loc[j]` for `j in 0..287`, so it silently assumes the last 3 days contain every 5-min slot.
8. `op_rate` in `select_policy*` is a constant 0.75 and the ODU low pressure is treated as the control variable, whereas the Full-DRL repo controls **high** pressure — the two repos are not interchangeable without re-mapping the action.
9. The HVAC-stage fits have low R² (0.27 / 0.24) and `Aac` for ODU#2 sits on its bound; the design document itself flags the high-pressure ΔT formula as unvalidated. Treat the simulated energy ranking as relative, not absolute.
10. Unused imports: `ctREFPROP` in the 2R2C and `make_order` notebooks, `traceback`, `datetime` (module) alongside `from datetime import datetime`.
11. No input data, DB, or REFPROP are included, so none of the notebooks run from a fresh clone (see §3).

---

## 8. Running the summer loop

```bash
# 0. once: identify the building/HVAC model (needs ODU#N_5min_v2_weekend_avg.xlsx next to the notebook)
jupyter nbconvert --to notebook --execute "Summer(Cooling mode)/2R2C_ODU1_2W5P_0817.ipynb"
#    → copy the printed parameters into the constants cell of select_policy.ipynb

# 1. weekly (scheduler): choose the policy coefficients
jupyter nbconvert --to notebook --execute "Summer(Cooling mode)/select_policy.ipynb"       # → C:/KHU_DRL/1st_policy.csv
jupyter nbconvert --to notebook --execute "Summer(Cooling mode)/select_policy_odu2.ipynb"  # → C:/KHU_DRL/2nd_policy.csv

# 2. every 10 minutes (scheduler): write the target low pressure
jupyter nbconvert --to notebook --execute "Summer(Cooling mode)/make_order.ipynb"          # → odu1_drl_policy
jupyter nbconvert --to notebook --execute "Summer(Cooling mode)/make_order_odu2.ipynb"     # → odu2_drl_policy
```

Prerequisites: `C:/KHU_DRL/NED_SQLite.db` being populated by the BMS logger, `C:/KHU_DRL/output/` existing, and REFPROP installed at `C:/Refprop_dll`.
