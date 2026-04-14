# TwinFrame · Hardware Abstraction Layer (v0.1)

Since physical hardware is unavailable during development, actuators protect their outputs using the global `SimulationMode` flag.

This pattern ensures that the component logic (states, timers, interlocks) executes fully, while physical output assignment is blocked at the last execution stage.

```pascal
// Bypass pattern in Actuators
METHOD PROTECTED DoStart : BOOL

// HAL intervention point (last-mile output protection)
IF NOT GVL_Components.SimulationMode THEN
    IO.Q_Start := TRUE; // Physical write only occurs in real mode
END_IF

// Logical execution continues regardless of hardware presence
DoStart := TRUE;
```
## Benefits
* **Software Validation:** State transitions (e.g. STATE_BUSY) execute fully even without physical outputs.
* **Safety:** Prevents unintended actuation during software-only testing or commissioning.
* **Consistency:** Same codebase is used in simulation and physical deployment.

## Input Abstraction

For sensors, the HAL allows the component to bypass physical inputs and use forced values for simulation and validation of thresholds and alarm logic.

```pascal
// Input abstraction layer (Sensors)
IF GVL_Components.SimulationMode THEN
    _RawValue := _SimValue;          // Forced simulation input
ELSE
    _RawValue := _IO.PhysicalInput;  // Real hardware input
END_IF
```

## Centralized Control
Instead of toggling simulation per component, the HAL is controlled globally. This ensures a consistent execution context across the entire system.

```pascal
// In the Main program (MAIN)
GVL_Components.SimulationMode := TRUE; // Enables "Office/Development" mode
```

| Context | SimulationMode | Behavior |
|:-----|:-----|:-----|
| **Development** | TRUE  | Forced inputs, blocked physical outputs. |
| **Commissioning** | FALSE  | Full control of real hardware.|

## Design Decisions (v0.1)

* **Last-Mile Bypass:** The simulation check is placed at the final output assignment stage to maximize test coverage of all internal logic.
* **No Mock Objects:** No mirror classes or simulation doubles are required. The same functional blocks are used for both simulation and deployment.
* **Simplicity First:** At this stage, complexity is intentionally avoided. A single global flag provides the most deterministic and maintainable solution.

---

## Current Limitations (v0.1)

* No modeling of physical dynamics (latency, inertia, noise, or signal degradation)
* No timing discrepancies between simulated and real hardware
* No fieldbus or I/O scan delay emulation
* Pure logical execution only

## Summary

This HAL is not intended to be a full digital twin system.
It is an early-stage execution abstraction layer designed to validate:

* Control logic correctness
* State machine behavior
* Alarm propagation
* System orchestration

Future versions will extend this layer toward realistic simulation and commissioning support.


