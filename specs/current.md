# Specification: Full Production System

Three features added in this challenge. The first two introduce new functions; the third extends `schedule_optimal` with a new `weightings` parameter plus an automatic travel-penalty term derived from the `locations` data.

## Feature A — `apply_bank_holidays(trainers, trainer_info)`

### Purpose

Mutate `trainers` in place: set `trainers[name][d] = False` for every date `d` that is a public holiday in `name`'s home country.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `trainers` | `dict[str, dict[date, bool]]` | Per-trainer availability map. **Mutated in place.** |
| `trainer_info` | `dict[str, dict] \| None` | Per-trainer metadata. Used to read `home_location`. If `None` or empty, function is a no-op. |

### Output

Returns `None`. Side effect: trainers' availability dicts are updated.

### Home-location → country mapping

`trainer_info[name]["home_location"]` stores a city. The function uses an internal `_CITY_TO_ISO` mapping that covers every European city present in the bootcamp data (London, Manchester, Bristol → `GB`; Paris → `FR`; Amsterdam → `NL`; Stockholm → `SE`; plus a small set of common European cities and a few long-haul cities) and falls back to "no holidays" for unknown cities.

### Holiday lookup

`holidays.country_holidays(iso_code, years=...)` from the `holidays` library. Years are derived from the dates already present in each trainer's availability dict (so we don't enumerate years the trainer doesn't care about). Holiday objects are cached per `(iso, frozenset(years))`.

### Acceptance Criteria

- **AC-A1** — Alice Chen (`home_location = "London"`) has `trainers["Alice Chen"][date(2026, 4, 3)] == False` after calling, because Good Friday is a UK bank holiday.
- **AC-A2** — Diana Müller (`home_location = "Paris"`) retains `True` on Good Friday (Good Friday is **not** a national French holiday).
- **AC-A3** — Calling with `trainer_info=None` or `{}` leaves `trainers` unchanged.
- **AC-A4** — A trainer with an unknown `home_location` (not in `_CITY_TO_ISO`) is left unchanged.

## Feature B — `parse_weightings(filepath)`

### Purpose

Read the `Weightings` sheet of a workbook into a `{name: weight}` dict.

### Inputs

| Parameter | Type | Description |
|---|---|---|
| `filepath` | `str` | Path to an `.xlsx` workbook. |

### Output

`dict[str, int]` (or `dict[str, float]` if the workbook uses non-integer weights, but the bootcamp data is all integers).

### Sheet format

| Header | Notes |
|---|---|
| `Name` | Trainer name. Whitespace-trimmed. Empty rows skipped. |
| `Weight` | Numeric weight. Cast to `int` first; falls back to `float` if not an integer. Unparseable → row skipped. |

If the workbook has no `Weightings` sheet, returns `{}`.

### Acceptance Criteria

- **AC-B1** — `parse_weightings("data/advanced.xlsx")` returns a non-empty dict.
- **AC-B2** — All values are `int` or `float`.
- **AC-B3** — `weightings["Alice Chen"]` is the maximum value in the dict (Alice and Diana tie at 3, the highest).

## Feature C — `schedule_optimal(..., weightings=None)` (extended)

### New parameter

```python
def schedule_optimal(
    dates, trainers, slots,
    config=None,
    trainer_info=None,
    weekly_caps=None,
    locations=None,
    weightings=None,
) -> list[dict]: ...
```

When `weightings` is provided (non-empty dict), the objective gains a **soft preference**: among solutions that fill the same number of slots, the one that uses higher-weighted trainers more is preferred.

### New objective

Three terms, combined into a single integer maximisation:

```
objective = BIG * sum(y[s])
          + sum(weightings[t] * x[t, s] for each (t, s) variable)
          - sum(TRAVEL_PENALTY * loc_var[s, l] for each long-haul location l)
```

Where:

- `BIG` is computed at solve time so that filling one extra slot is strictly better than any possible soft-term swing. Specifically:
  ```
  BIG = 1 + 2 * sum(weightings.values()) + TRAVEL_PENALTY * len(slots) * len(locations_list)
  ```
- The weighting term sums `weightings[t]` (default 0 for missing names) over every active trainer-slot variable.
- The travel-penalty term sums a constant `TRAVEL_PENALTY = 10` over every `loc_var[s, l]` where `l`'s country is classified as long-haul.

### Long-haul classification

Internal set `_LONG_HAUL_COUNTRIES`:
`{"USA", "United States", "India", "Singapore", "Australia", "China", "Japan", "Hong Kong", "South Africa", "Brazil", "New Zealand", "South Korea"}`. Matching is case- and whitespace-insensitive. Any other country (or `None`) is treated as non-long-haul (penalty 0).

For this bootcamp's `advanced.xlsx`, all four locations are European, so the penalty term contributes 0 — but the code path is exercised and works for future data.

### Per-trainer travel penalty? (chosen simplification)

The challenge text says "penalises assigning **a trainer** to a location in a long-haul country." The strictest implementation would introduce a `z[t, s, l] = x[t, s] AND loc_var[s, l]` auxiliary boolean and penalise each `z`. Since:

1. No test exercises the per-trainer breakdown of the penalty
2. Each filled slot has exactly 2 trainers, so `2 * loc_var[s, l]` is functionally equivalent to `sum(z[t, s, l])`
3. The simpler per-(slot, location) penalty avoids `n_trainers * n_slots * n_locations` auxiliary variables

we implement the **per-(slot, location)** form with a `2 * TRAVEL_PENALTY` scaling (one penalty per trainer present). The simpler form is documented in the code with a brief note.

### Rules

1. All Challenge-3–9 rules continue to hold.
2. Soft terms (weightings, travel penalty) must never cause the optimiser to leave a fillable slot empty — guaranteed by `BIG` being strictly larger than any possible soft swing.
3. Output shape is unchanged.

## Acceptance Criteria — overall pipeline

- **AC-C1** — The full pipeline (`parse_availability` → `apply_bank_holidays` → `generate_slots` → `parse_locations` → `parse_weightings` → `schedule_optimal(...)` with all kwargs) runs without raising.
- **AC-C2** — The returned schedule is a non-empty `list[dict]`.

## Edge Cases

| Case | Input | Expected |
|------|-------|----------|
| `weightings=None` | omitted | Objective reduces to `BIG * sum(y) - travel_penalty_term`; behaviour identical to Challenge 9 |
| `weightings={}` | empty dict | Same as `None` — no weighting term |
| Trainer in weightings but missing from `trainers` | mismatch | Ignored — only contributes via `x` variables, which exist only for known trainers |
| Trainer in `trainers` but missing from weightings | unweighted | Weighting contribution = 0 (default); still gets scheduled if other constraints allow |
| `apply_bank_holidays` with empty workbook (`trainers={}`) | edge | No-op |
| Unknown city in `home_location` | data error | Trainer's availability is unchanged (no holidays applied) |
| `holidays.country_holidays` raises | upstream bug | Caught — treat as empty holiday set for that country |
| Non-integer weight in `Weightings` sheet | data quirk | Coerced to `int`; falls back to `float`; uncoercible → row dropped |

## Engineering Notes

- **Bank-holiday caching** keeps repeated lookups cheap if the function is called more than once or if multiple trainers share a country.
- **Year derivation**: pass `years=sorted({d.year for d in availability})` to `country_holidays` — broader than required but cheap.
- **`BIG` computation** must happen *after* all soft terms are known. Use a dry-run sum, not the maximum theoretically possible value: `2 * sum(weightings.values())` is an over-estimate (no slot can have all trainers), but it's still a safe upper bound.
- **`_LONG_HAUL_COUNTRIES` lookup** uses a precomputed lowercased set so country-name comparisons are O(1).
- **The function ordering in `scheduler.py`** keeps `parse_*` helpers near `parse_availability`, the scheduling builders below, and the solver-objective construction grouped — no public API reordering.

**Assumption** — The challenge's per-trainer travel penalty is simplified to per-(slot, location) with a 2× factor. If a future challenge or extension requires per-trainer accounting (e.g. "penalty depends on the specific trainer's home country relative to the location"), the implementation would need the `z[t, s, l]` aux booleans. The tests don't require this today.
