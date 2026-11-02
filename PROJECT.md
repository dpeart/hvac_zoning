# HVAC Zoning System — Home Assistant Package

## Project Overview

This is a **zoned HVAC control system** for a two-story home, implemented as a Home Assistant *(HA)* configuration package. It opens and closes motorized damper relays (via ESPHome relay boards) per room/zone to direct conditioned air only where it is needed, based on each zone's temperature error against a central thermostat setpoint.

The system has two operational modes:

1. **Production mode** — reads real zone temperatures and the real thermostat state, computes per-zone demand, and drives the dampers accordingly.
2. **Test mode** — replaces sensor inputs with manual sliders (per-zone temps, target temp, action) so the whole demand→register→relay pipeline can be exercised and validated without physical sensors, at a fast 10-second cadence.

> **Why test mode exists:** the owner wanted to validate the automation logic (demand math, minimum-airflow allocation, proportional relay control) before trusting it in production, using a dashboard as a test bench.

---

## Repository Layout

```
packages/hvac_zoning/
├── upstairs/
│   ├── hvac_upstairs_globals.yaml       # Upstairs tuning knobs (deadband, sensitivity, weights, loop interval, min airflow)
│   ├── hvac_upstairs_automations.yaml  # Control loop automation
│   ├── hvac_upstairs_inputs.yaml       # Zone helpers: names, temp/occupancy entity wiring
│   ├── hvac_upstairs_sensors.yaml      # Template sensors: temp, error, demand, raw registers, total
│   └── hvac_upstairs_test.yaml         # Test-mode inputs (define entities used by dashboard)
├── downstairs/
│   ├── hvac_downstairs_automations.yaml  # Downstairs control loop
│   ├── hvac_downstairs_globals.yaml      # Downstairs tuning knobs (own set, prefixed hvac_downstairs_)
│   ├── hvac_downstairs_inputs.yaml       # Downstairs zone helpers
│   ├── hvac_downstairs_sensors.yaml      # Downstairs template sensors
│   └── hvac_downstairs_test.yaml         # Downstairs test-mode inputs
└── dashboards/
    ├── hvac_zoning_dashboard.yaml        # YAML-mode dashboard: Upstairs + Downstairs views (single file)
    ├── hvac_upstairs_dashboard.yaml      # Extracted upstairs view (source reference)
    └── hvac_downstairs_dashboard.yaml    # Extracted downstairs view (source reference)
```

**Loading mechanism:** `configuration.yaml` loads this project's packages **explicitly**, one `!include` per file (no recursive glob), e.g.:
```yaml
homeassistant:
  packages:
    pkg_upstairs_globals: !include packages/hvac_zoning/upstairs/hvac_upstairs_globals.yaml
    pkg_upstairs_automations: !include packages/hvac_zoning/upstairs/hvac_upstairs_automations.yaml
    # ... one entry per package file
```

This is deliberate: `!include_dir_named packages` would recursively glob **every** `.yaml` under `packages/` into a package definition, which breaks on the dashboard files (top-level `title:`/`views:` are not valid package keys) and silently collides on duplicate basenames across subfolders. Loading explicitly keeps `dashboards/` (and any other non-package YAML) safe inside the repo. Package names (`pkg_*`) are arbitrary but must be unique.

**Dashboard:** the single YAML-mode dashboard `hvac_zoning_dashboard.yaml` is registered in `configuration.yaml` and loaded from disk on every restart (no pasting, no drift). The two per-floor files are kept as per-floor sources for reference.

> **YAML-mode dashboard registration:** under `lovelace: dashboards:`, the **dictionary key is the URL path** and Home Assistant requires it to contain a hyphen (`-`). Use `hvac-zoning` (not `hvac_zoning`), e.g.:
> ```yaml
> lovelace:
>   mode: yaml
>   dashboards:
>     hvac-zoning:
>       mode: yaml
>       title: HVAC Zoning
>       filename: packages/hvac_zoning/dashboards/hvac_zoning_dashboard.yaml
> ```

**Entity naming conventions:**
- Upstairs: `input_number.hvac_test_zone_1_temp`, `sensor.upstairs_zone_1_error`, `input_boolean.hvac_test_mode`, etc.
- Downstairs: everything is prefixed `hvac_downstairs_*` (e.g. `hvac_downstairs_test_mode`, `sensor.downstairs_zone_1_error`).
- The prefix separation is essential — both floors coexist in the same HA instance, so **entity IDs must not collide**.

---

## Zone & Relay Topology

### Upstairs (board: `switch.iotmug_16_relay_board_relay_N`)

| Zone | Register count (input_number) | Relays |
|------|-------------------------------|--------|
| Z1 Stairs | 1 | relay 1 |
| Z2 Bed 1 | 3 | relays 2, 3, 4 |
| Z3 Bed 2 | 2 | relays 5, 6 |
| Z4 Bed 3 | 1 | relay 7 |
| Z5 Master Bed | 3 | relays 8, 9, 10 |
| Z6 Master Bath | 3 | relays 11, 12, 13 |
| **Total** | **13** | 13 relays |

The 16-relay board has relays 14–16 unused upstream.

### Downstairs (board: `switch.iotmug_2_relay_board_relay_N` — placeholder prefix, verify)

| Zone | Register count (input_number) | Relays |
|------|-------------------------------|--------|
| Z1 Kitchen | 2 | relays 1, 2 |
| Z2 Dining | 2 | relays 3, 4 |
| Z3 Living | 3 | relays 5, 6, 7 |
| Z4 Office | 2 | relays 8, 9 |
| Z5 Stairs | 1 | relay 10 |
| **Total** | **10** | 10 relays |

Downstairs relay entity prefix is `iotmug_2_` (placeholder — verify against the ESPHome device).

---

## Core Concepts & Math

### 1. Temperature Error (per-zone template sensor `*_error`)

```
error = |current_temp − target_temp|  (signed by action direction)
```

- **Cooling:** `error = current − target` (positive when room is hotter than setpoint).
- **Heating:** `error = target − current` (positive when room is colder than setpoint).
- **Idle/off:** error = 0.
- **Test mode:** `target` = `hvac_test_target_temp`, `current` = per-zone test temp slider, `action` = `hvac_test_action`.
- **Production:** `target` = thermostat `temperature` attribute, `current` = zone's configured temp sensor, `action` = thermostat `hvac_action` attribute.

### 2. Demand (per-zone template sensor `*_demand`)

```
raw_demand = (error − deadband) / sensitivity     # 0 if error ≤ deadband
raw_demand = min(raw_demand, 1.0)                 # capped
demand     = raw_demand × (weight_occupied if occupied else weight_unoccupied)
```

- **Deadband** (`hvac_zone_deadband`): error below this produces zero demand (currently 0.5°F default, tuned down to 0.1 during testing).
- **Sensitivity** (`hvac_zone_sensitivity`): error→demand scaling. Lower = more aggressive.
- **Weights** (`hvac_zone_weight_occupied` / `hvac_zone_weight_unoccupied`): set both to 1.0 so occupancy doesn't dampen airflow (owner may never add occupancy sensors).
- **Important pitfall:** raw demand is *capped at 1.0* before weighting, so an unoccupied zone with weight 0.2 can never exceed 0.2 demand — a 3-register zone caps at `ceil(0.2×3) = 1` register regardless of error. This is why the weights defaulting to 0.2 caused "master bath only opens 1 damper" confusion.

### 3. Raw Registers (per-zone template sensor `*_raw_registers`)

```
registers = ceil(demand × zone_register_count)
```

### 4. Total Demand (`sensor.*_total_demand`)

```
total = Σ raw_registers over zones
```

### 5. Minimum Airflow Allocation (automation variables)

The `hvac_min_airflow` input (register equivalents, default 1) protects the blower/furnace. When `total_demand < min_airflow`, the **deficit** is allocated across zones in **fixed priority order** (stairs first — smallest zones first), up to each zone's physical capacity:

```
deficit       = max(min_airflow − total_demand, 0)
z1_extra      = min(deficit, max(cap_1 − r1, 0))
z2_extra      = min(deficit − z1_extra, max(cap_2 − r2, 0))
... (chained)
rN_eff        = rN + zN_extra   # effective register count used by relays
```

Capacities: upstairs Z1=1, Z2=3, Z3=2, Z4=1, Z5=3, Z6=3. Downstairs Z1=2, Z2=2, Z3=3, Z4=2, Z5=1.

### 6. Proportional Relay Control (automation)

Each relay is toggled **individually** so the number of open relays in a zone matches the *effective* register count:

```
open relay N_i  iff  rN_eff ≥ i
close relay N_i otherwise (default)
```

(Relay OFF = damper OPEN = airflow; relay ON = damper CLOSED.)

---

## Automation: Control Loop (`automation.upstairs_hvac_control_loop` / `automation.hvac_downstairs_zone_control_loop`)

### Trigger & Conditions

```yaml
trigger:   time_pattern seconds: "/10"
condition:
  - is_state('input_boolean.hvac_*_test_mode', 'on')
    OR  (timestamp % loop_interval == 0)      # loop_interval from input_number (default 90s)
  - is_state('input_boolean.hvac_*_zoning_enabled', 'on')
```

- **Test mode:** runs every 10 s (all triggers pass, no modulo gate).
- **Production:** runs when the current epoch second is a multiple of the configured interval (e.g. 90 → fires at :00, :90 of each 2-minute window). `mode: single` prevents queueing.

### Action Flow

1. **Compute variables** (action, raw registers rN, total_demand, min_airflow, deficit, extras, effective rN_eff).
2. **If action in [heating, cooling]:**
   - Log active loop summary.
   - If deficit > 0, log min-airflow warning.
   - Per relay (all zones): open/close based on `rN_eff ≥ i`.
3. **Else (idle/off):** failsafe — open ALL dampers (turn_off all relays) + warning log.

---

## Sensors (Template) — Per Floor

For each zone N:

| Sensor | ID pattern | Meaning |
|--------|-----------|---------|
| Error | `sensor.upstairs_zone_N_error` / `sensor.downstairs_zone_N_error` | Signed temp error (°F) in action direction |
| Demand | `sensor.upstairs_zone_N_demand` / `sensor.downstairs_zone_N_demand` | 0–1.0 demand value (weighted) |
| Raw Registers | `sensor.upstairs_zone_N_raw_registers` / `sensor.downstairs_zone_N_raw_registers` | Effective register count (0–cap) |
| Total | `sensor.upstairs_total_demand` / `sensor.downstairs_total_demand` | Sum of raw registers |

The error sensors are test-mode aware: they substitute test inputs when `input_boolean.hvac_*_test_mode` is on. Demand and registers derive from error, so they follow automatically.

---

## Input Helpers (Knobs & Wiring)

### Global tuning (per floor)

| Input | Purpose | Default |
|-------|---------|---------|
| `input_number.hvac_zone_deadband` | Error threshold below which no demand (°F) | 0.5 (tested at 0.1) |
| `input_number.hvac_zone_sensitivity` | Error→demand scaling | 1.0 |
| `input_number.hvac_zone_weight_occupied` | Occupied multiplier | 1.0 |
| `input_number.hvac_zone_weight_unoccupied` | Unoccupied multiplier | 1.0 |
| `input_number.hvac_zone_loop_interval` | Production loop seconds | 90 |
| `input_number.hvac_min_airflow` | Minimum register equivalents | 1 |
| `input_boolean.hvac_zoning_enabled` | Master on/off | off |

(Downstairs equivalents all prefixed `hvac_downstairs_`; note `hvac_zone_loop_interval` upstairs lives in `upstairs/hvac_upstairs_globals.yaml`, downstairs in its own `downstairs/hvac_downstairs_globals.yaml`.)

### Test bench inputs

| Input | Purpose |
|-------|---------|
| `input_boolean.hvac_test_mode` | Toggle test mode (also gates dashboard conditionals) |
| `input_select.hvac_test_action` | heating / cooling / idle / off |
| `input_number.hvac_test_target_temp` | Test setpoint (°F) |
| `input_number.hvac_test_zone_N_temp` | Per-zone current temp slider (°F) — 0.1°F steps |

The legacy `input_number.hvac_test_zone_N_registers` (0–13 sliders) still exist but are **no longer read** by the automation; they were replaced by the temperature-driven test bench. Leave them unused.

### Zone wiring (input_text)

| Input | Purpose |
|-------|---------|
| `input_text.hvac_zone_N_name` | Display name |
| `input_text.*_zone_N_temp_entity` | **Entity ID of the zone's temperature sensor** — this is how you assign sensors per zone |
| `input_text.*_zone_N_occ_entity` | Occupancy binary sensor (placeholders for most zones) |

> **Assigning temperature sensors:** set the `input_text` state in Developer Tools → States or via the dashboard's "Zone Temperature Sensors" card (production mode). Editing `initial:` in YAML does NOT update already-created helpers.

---

## Dashboard (Lovelace)

The dashboard (`dashboards/hvac_zoning_dashboard.yaml`, path `hvac-zoning`) has two views — Upstairs and Downstairs. Each view has **conditionally visible cards** keyed on the floor's test-mode boolean:

| Card | Visibility | Contents |
|------|-----------|----------|
| Thermostat + Enable + Test Mode | Always | Master controls |
| Zone math grid (error/demand/registers per zone) | **Always** | Test-aware sensors, so it shows live demand in both modes |
| History graph | Production only (`state_not: 'on'`) | Total demand + per-zone registers over 1h |
| Zone Temperature Sensors | Production only | `input_text` wiring helpers (click to assign sensors) |
| Test bench inputs | Test only (`state: 'on'`) | Action, target, 5–6 zone temp sliders, "Run Zone Loop Now" button |
| Tuning | Always | Min airflow, sensitivity, deadband, weights, loop interval |
| Relay glance | Always | Per-relay damper state (ON=closed, OFF=open) |
| Gauge | Always | Total demand vs max (13 upstairs / 10 downstairs) |
| Last run + logbook | Always | Automation entity, `last-triggered`, recent log |

**"Run Zone Loop Now" button** uses `automation.trigger` with `skip_condition: true` — required, otherwise the modulo condition silently aborts manual runs.

**Known limitation:** conditional cards render unconditionally while the dashboard is in **edit mode**; exit edit mode to see the test/production switching.

---

## Deployment / Onboarding Steps

1. **Load packages:** the HVAC package files load on restart via the explicit `!include` entries in `configuration.yaml` under `homeassistant.packages`. Restart or "Reload all YAML" after editing entity definitions.
2. **Dashboard:** the YAML-mode dashboard `dashboards/hvac_zoning_dashboard.yaml` is registered under `lovelace: dashboards: hvac-zoning:` in `configuration.yaml` and loads from disk on restart. Edits to the `.yaml` need a restart (or Dev Tools → YAML → reload Lovelace).
3. **Assign temp sensors:** in production mode, use the "Zone Temperature Sensors" card (or Developer Tools → States) to point each `input_text.*_temp_entity` at the real sensor entity ID. Defaults: stairs/masterbed → thermostat current-temp sensor; others → `sensor.placeholder_*`.
4. **Verify zones:** confirm the thermostat climate entity matches the floor (`climate.upstairs_nativezone` / `climate.downstairs_nativezone`), and relay entity prefixes (`switch.iotmug_16_relay_board_relay_N` / `switch.iotmug_2_relay_board_relay_N`).

---

## Testing Workflow (Test Mode)

1. Toggle **Test Mode** ON on the dashboard.
2. Set **Test Action** (heating/cooling).
3. Set **Test Target** and per-zone **temp sliders** (0.1°F steps).
4. Loop runs every 10 s; watch zone math cards update, then relay glance reflect dampers.
5. Use **Run Zone Loop Now** for an immediate single pass.
6. Watch `system_log` for `Active Loop` (info) and `Minimum Airflow Enforced` (warning) messages.

### Demand → Registers quick reference (occupied, weight 1.0, sensitivity 1.0)

| Error (°F) | Demand | Registers (3-reg zone) |
|-----------|--------|------------------------|
| ≤ deadband | 0 | 0 |
| +0.1 above db | ~0.1 | 1 |
| +0.5 above db | ~0.5 | 2 |
| +0.9 above db | ~0.9 | 3 |

(With deadband 0.5/sensitivity 1.0: error 0.6→1 reg, 1.1→2 regs, 1.6→3 regs.)

---

## Troubleshooting & Gotchas

- **Min-airflow didn't open enough dampers:** allocation only opens zones in priority order stairs→…→up to capacity. If deficit is consumed by early zones, later zones stay closed. Check the `Minimum Airflow Enforced` warning log for the effective demands.
- **Unoccupied zones cap at 1 register:** caused by `min(raw, 1.0)` before the 0.2 unoccupied weight. Fix: raise `hvac_zone_weight_unoccupied` to 1.0 (both floors now default 1.0).
- **`input_number` "expected a mapping/float" errors:** means an entity body got indented under another entity (nested instead of top-level). Check indentation in `hvac_*_test.yaml`.
- **Dashboard "expected str at entity_ids[N]" history error:** a card got nested inside the history-graph's `entities:` list (indentation bug). Keep conditional cards at the top level of `cards:`.
- **Changing `initial:` in YAML doesn't update existing helpers** — the value in `.storage` wins. Use the dashboard slider or Developer Tools → States, or delete the helper and reload.
- **Entity ID from automation alias:** YAML automations get `automation.<slugified_alias>` (e.g. `alias: "HVAC Upstairs: Zone Control Loop"` → `automation.hvac_upstairs_zone_control_loop`). The dashboard must target exactly this.
- **`automation.trigger` without `skip_condition: true`** will silently no-op due to the modulo condition — always pass `skip_condition: true` for manual test runs.
- **Test sliders step:** keep 0.1°F so you can hit register thresholds; 0.5°F steps jump straight to max registers.
- **BOM/encoding:** files were saved with a UTF-8 BOM at the top; HA tolerates it, but re-saving as UTF-8 without BOM avoids subtle issues. Watch for `Â°F` mojibake from encoding round-trips in PowerShell (`Set-Content -Encoding utf8` writes a BOM on Windows PowerShell).
- **behind the scenes — the inherited bug history:** the original upstream automation opened ALL of a zone's relays on any demand (all-or-nothing), and the old min-airflow safeguard opened only Zone 1 then immediately got overridden by Zone 1's own block. Both were fixed: proportional per-relay control and deficit allocation via `rN_eff`.

---

## Future Work / Open Items

- [ ] Verify downstairs relay board entity prefix (`iotmug_2_` is a placeholder).
- [ ] Assign real temp sensors for Bed 3 / Master Bath (upstairs Z4/Z6) and all downstairs zones.
- [ ] Occupancy: if real occupancy sensors are never installed, the weight inputs (both 1.0) make occupancy a no-op — safe to leave.
- [ ] `hvac_test_zone_N_registers` legacy sliders could be removed once the temperature-driven bench is confirmed stable.
- [ ] Consider making min-airflow allocation occupancy-aware (favor occupied zones) if sensors are added.
