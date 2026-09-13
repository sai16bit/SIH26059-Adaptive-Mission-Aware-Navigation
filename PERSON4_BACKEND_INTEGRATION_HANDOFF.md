# SIH26059 — Person 4 Backend / Integration Handoff

## What Person 4 provides

Person 4 consumes real Person 2 CV detections, Person 3 trajectory predictions, and Person 5 sea-ice information to produce a mission-aware, safety-validated route and replan it when new trajectory/environmental information arrives.

## Recommended integration flow

```text
Person 2 CV detections
        |
        v
Current iceberg risk
        |
Person 3 trajectory predictions
        |
        v
Future trajectory + uncertainty risk
        |
Person 5 sea-ice concentration
        |
        v
Combined risk surface + hard NO-GO map
        |
        v
Mission-aware route planner
        |
        +--> required scientific waypoint(s)
        +--> optional waypoint(s)
        +--> risk / distance / ETA / mission constraints
        |
        v
Route safety validator
        |
        v
Backend/API response
        |
        +--> new environmental update
                 |
                 v
             automatic replanning
```

## Files to provide

1. `SIH26059_Adaptive_Mission_Aware_Navigation_REAL_TEAM_DATA_EXECUTED.ipynb`
   - Full Person 4 prototype and demonstration.

2. `cv_iceberg_detections_used.csv`
   - Real Person 2 CV detections used by the notebook.

3. `trajectory_predictions_person4.csv`
   - Real Person 3 prediction output consumed by Person 4.

4. `trajectory_update_used.csv`
   - The real Person 3 prediction selected for the adaptive-replanning demonstration.

5. `risk_grid_real_data.csv`
   - Generated risk/cost grid. Useful for frontend visualization/debugging, but the backend should be able to regenerate it from current inputs rather than treating this CSV as the source of truth.

6. `adaptive_mission_results_real_data.json`
   - Generated summary of before/after route metrics and USP evidence.

## Person 3 prediction contract

Required fields:

```text
iceberg_id
prediction_time
horizon_hours
predicted_latitude
predicted_longitude
uncertainty_km
```

Missing `uncertainty_km` must NOT be interpreted as zero uncertainty. The Person 3 handoff states that 6h/12h predictions may have unavailable uncertainty, while trained XGBoost models are currently available for 24h/48h.

## Person 2 CV contract

The Person 4 risk layer uses:

```text
iceberg_id
latitude
longitude
area_km2
confidence
timestamp
```

Optional detection metadata can be retained, but the above fields are the integration contract.

## Route request from backend

Conceptually:

```json
{
  "start": {"latitude": -66.20, "longitude": 132.00},
  "goal": {"latitude": -65.10, "longitude": 138.00},
  "mission": {
    "required_waypoints": [
      {"id": "SURVEY-A", "latitude": -65.50, "longitude": 135.00, "service_hours": 4.0, "priority": 1.0}
    ],
    "optional_waypoints": [
      {"id": "OBS-B", "latitude": -65.20, "longitude": 137.00, "service_hours": 2.0, "priority": 0.7}
    ],
    "max_eta_hours": 60.0,
    "distance_budget_km": 1000.0
  }
}
```

## Route response

The backend should expose at least:

```json
{
  "route": [
    {"latitude": -66.20, "longitude": 132.00},
    {"latitude": -65.50, "longitude": 135.00},
    {"latitude": -65.10, "longitude": 138.00}
  ],
  "distance_km": 537.29,
  "travel_eta_hours": 24.18,
  "mission_eta_hours": 24.18,
  "average_risk": 0.062,
  "max_risk": 0.311,
  "route_safe": true,
  "mission_feasible": true,
  "science_stops": ["SURVEY-A"],
  "replanned": true
}
```

The exact numeric values above are demonstration values from the current supplied dataset; the backend should calculate them dynamically.

## Adaptive update endpoint / event

When a new Person 3 prediction arrives, do NOT move an iceberg artificially. Pass the new prediction through the risk engine, rebuild the risk/NO-GO surface, rerun mission planning, validate the new route, and return whether the route changed.

Suggested event payload:

```json
{
  "type": "trajectory_update",
  "iceberg_id": "c32",
  "prediction_time": "2023-03-07T00:00:00",
  "horizon_hours": 48,
  "predicted_latitude": -66.05,
  "predicted_longitude": 132.75,
  "uncertainty_km": 5.0
}
```

## Important integration note

The supplied Person 2 CV IDs and Person 3 trajectory IDs/timestamps are not currently synchronized one-to-one. Do NOT fabricate an ID match. Treat Person 3 predictions as a trajectory stream and use explicit spatial/temporal association when the upstream teams provide synchronized operational data.

## Safety rules that must remain server-side

- A route cannot contain a hard NO-GO cell.
- A required scientific waypoint must be reachable for the mission to be feasible.
- Optional scientific waypoints may be skipped if they make the mission unsafe or violate constraints.
- Missing trajectory uncertainty is not zero uncertainty.
- Route smoothing is display-only and must be line-of-sight checked against NO-GO cells.
- The backend should revalidate the route after every environmental update.

## What the frontend needs

The frontend does not need to implement A*. It should receive:

- route coordinates
- current risk grid / risk tiles if visualization is required
- iceberg markers
- trajectory prediction markers/uncertainty
- scientific waypoint markers
- route safety status
- mission feasibility
- ETA/distance/risk metrics
- `replanned` flag and update timestamp

## USP evidence

### USP 1 — Mission-Aware Navigation

The planner does not simply solve start-to-goal. It considers required/optional scientific waypoints, waypoint priority, service time, route risk, distance budget, ETA and hard safety constraints.

### USP 2 — Self-Updating / Continuous Replanning

A real Person 3 trajectory prediction changes the risk surface, triggers complete mission re-evaluation, and can produce a different validated route.

Combined:

**Adaptive Mission-Aware Navigation = new environmental data → risk recalculation → complete mission re-evaluation → automatic safe replanning.**
