# TwinFrame · Interface Reference

TwinFrame applies **Interface Segregation** as a hard architectural rule. Every interface in the framework owns exactly one responsibility. No component is forced to implement methods it does not need.

The practical consequence is that process logic never holds a reference to a concrete block — only to the interface that exposes the behavior it needs. A supervisory sequence that only needs to start and stop a pump holds `I_StartStop`, not `FB_Pump`. A diagnostic monitor holds `I_Diagnostic`, not `FB_ComponentBase`. This keeps every layer of the system independently replaceable and independently testable.

There are nine interfaces in the framework, organized into three groups:

**Lifecycle contracts** — govern how components are controlled and observed: `I_Component`, `I_Commandable`, `I_StartStop`, `I_ErrorChecker`, `I_Diagnostic`.

**Signal contracts** — govern how sensor data is accessed: `I_SensorBase`, `I_AnalogSensor`, `I_DigitalSensor`.

**Orchestration contracts** — govern how managers interact with the rest of the system: `I_ComponentManager`, `I_AlarmManager`.

---

## I_Component

The primary lifecycle interface. Every object in the TwinFrame hierarchy implements this interface. It is the densest interface in the framework because lifecycle management is the one responsibility that every component shares without exception.

**Implemented by:** `FB_ComponentBase` and all derived classes.

### Methods

| Method | Returns | Description |
|---|---|---|
| `Init` | `BOOL` | First-cycle initialization. Binds parameters, transitions to `STATE_IDLE` on success. |
| `Update` | `BOOL` | Cyclic entry point. Called every PLC cycle by `FB_ComponentManager.UpdateAll`. |
| `Reset` | `BOOL` | Fault recovery. Transitions from `STATE_ABORTED` or `STATE_ERROR` back to `STATE_IDLE`. |
| `Enable` | `BOOL` | Activates the component for operation. |
| `Disable` | `BOOL` | Deactivates the component. Transitions to `STATE_STOPPED`. |
| `Abort` | `BOOL` | Emergency stop. Immediately transitions to `STATE_ABORTING` regardless of current state. |
| `AcknowledgeAlarm` | `BOOL` | Marks active alarms on this instance as acknowledged. |

### Properties

| Property | Type | Access | Description |
|---|---|---|---|
| `State` | `E_ComponentState` | GET | Current PackML state machine state. |
| `Mode` | `E_ExecutionMode` | GET | Current execution mode (Automatic, Simulation, Test, etc.). |
| `Ready` | `BOOL` | GET | TRUE when component is in a state that accepts commands. |
| `Busy` | `BOOL` | GET | TRUE when a transition or operation is in progress. |
| `ID` | `UDINT` | GET | Unique component identifier assigned at initialization. |
| `HasAlarm` | `BOOL` | GET | TRUE if any active alarm exists on this component. |
| `HasFault` | `BOOL` | GET | TRUE if any active alarm has `IsFault = TRUE`. |
| `ActiveAlarmCount` | `UDINT` | GET | Number of currently active alarms. |
| `ActiveAlarms` | `ARRAY OF ST_Alarm` | GET | Full alarm array for this component. |

### Usage Pattern

```pascal
// Process logic holds the interface reference, not the concrete type
VAR
    _Pump : I_Component;
END_VAR

// Initialization
_Pump.Init();

// Cyclic check
IF _Pump.HasFault THEN
    _Pump.Abort();
ELSIF _Pump.Ready THEN
    // ready to receive commands
END_IF
```

---

## I_Commandable

Decouples external command dispatch from lifecycle logic. Allows a supervisory layer, HMI, or SCADA system to send typed commands without calling lifecycle methods directly. Commands are dispatched as `ST_Command` structures carrying the command type, execution flag, and timestamp.

**Implemented by:** `FB_ComponentBase` and all derived classes.

### Methods

| Method | Returns | Input | Description |
|---|---|---|---|
| `SendCommand` | `BOOL` | `Cmd : ST_Command` | Dispatches a typed command to the component's internal state machine. |

### Usage Pattern

```pascal
VAR
    _Pump    : I_Commandable;
    _Command : ST_Command;
END_VAR

// Build and dispatch a typed command
_Command.Cmd     := E_CommandType.CMD_START;
_Command.Execute := TRUE;
_Command.Timestamp := GVL_Components.CurrentDT;

_Pump.SendCommand(Cmd := _Command);
```

**Why this exists alongside `I_StartStop`:** `I_Commandable` is for structured command dispatch — sequences, supervisory systems, and SCADA layers that manage command history and timestamps. `I_StartStop` is for direct, immediate flow control — an interlock that needs to stop a pump in one line without building a command structure. Both are valid depending on the calling context.

---

## I_StartStop

Explicit flow control for actuators and functional units. Exposes `Start` and `Stop` as direct control verbs with a `Running` feedback property. Intentionally minimal — it owns only what is necessary for basic flow control.

**Implemented by:** `FB_ActuatorBase` and all derived actuator classes, `FB_PumpUnit`.

### Methods

| Method | Returns | Description |
|---|---|---|
| `Start` | `BOOL` | Commands the component to transition to `STATE_STARTING` → `STATE_RUNNING`. |
| `Stop` | `BOOL` | Commands the component to transition to `STATE_STOPPING` → `STATE_STOPPED`. |

### Properties

| Property | Type | Access | Description |
|---|---|---|---|
| `Running` | `BOOL` | GET | TRUE when the component is in `STATE_RUNNING`. |

### Usage Pattern

```pascal
VAR
    _Pump : I_StartStop;
END_VAR

// Direct flow control — no command structure needed
IF _StartCondition AND NOT _Pump.Running THEN
    _Pump.Start();
END_IF

IF _StopCondition AND _Pump.Running THEN
    _Pump.Stop();
END_IF
```

---

## 5. I_ErrorChecker

Isolates structured validation logic from state management. A component that performs multi-condition error checking before executing an action implements this interface to expose that validation as a callable contract.

**Implemented by:** `FB_SensorBase` and derived classes that require pre-execution validation.

### Methods

| Method | Returns | Description |
|---|---|---|
| `CheckErrors` | `BOOL` | Evaluates all error conditions. Returns TRUE if no blocking errors are present. |

### Usage Pattern

```pascal
VAR
    _Temp : I_ErrorChecker;
END_VAR

// Validate before commanding
IF _Temp.CheckErrors() THEN
    // safe to proceed
ELSE
    // error conditions present — hold sequence
END_IF
```

---


### Usage Pattern

```pascal
VAR
    _Component : I_Diagnostic;
    _Status    : ST_ComponentStatus;
END_VAR

_Status := _Component.GetStatus();

// Status is now OPC UA-serializable without additional mapping
IF _Status.Error THEN
    // handle error condition
END_IF
```

---

## I_SensorBase

The common contract for all sensor types. Defines validity and engineering unit metadata — the two properties that every sensor must expose regardless of signal type.

**Implemented by:** `FB_SensorBase`, `FB_AnalogSensor`, `FB_DigitalSensor`.

### Properties

| Property | Type | Access | Description |
|---|---|---|---|
| `Valid` | `BOOL` | GET | TRUE when the signal is within acceptable quality bounds. FALSE on wire-break, out-of-range, or hardware fault. |
| `Units` | `STRING` | GET | Engineering unit label (e.g., `'bar'`, `'L/min'`, `'°C'`). |

### Usage Pattern

```pascal
VAR
    _Sensor : I_SensorBase;
END_VAR

// Always check validity before using a sensor value
IF _Sensor.Valid THEN
    // safe to read signal
ELSE
    // signal quality compromised — hold dependent logic
END_IF
```

---

## I_AnalogSensor

Extends `I_SensorBase` with continuous signal access and runtime parameter configuration. The interface through which process logic reads scaled engineering values and through which cells configure threshold limits.

**Implemented by:** `FB_AnalogSensor`.  
**Extends:** `I_SensorBase`.

### Properties

| Property | Type | Access | Description |
|---|---|---|---|
| `Value` | `LREAL` | GET | Scaled engineering-unit value after gain, offset, and deadband processing. |

### Methods

| Method | Returns | Description |
|---|---|---|
| `SetParameters` | `BOOL` | Runtime threshold reconfiguration. Sets LoLo / Lo / Hi / HiHi limits without reinitializing the component. |

### Usage Pattern

```pascal
VAR
    _TempSensor : I_AnalogSensor;
END_VAR

// Configure thresholds at startup
_TempSensor.SetParameters(Lo := 10.0, LoLo := 5.0, Hi := 80.0, HiHi := 90.0);

// Read value in process logic
IF _TempSensor.Valid AND _TempSensor.Value > 75.0 THEN
    // approaching high threshold — take preventive action
END_IF
```

---

##  I_DigitalSensor

Extends `I_SensorBase` with discrete signal behavior. Exposes current state and edge detection, encapsulating debounce timing and transition logic internally so process logic never needs to implement it.

**Implemented by:** `FB_DigitalSensor`.  
**Extends:** `I_SensorBase`.

### Properties

| Property | Type | Access | Description |
|---|---|---|---|
| `Value` | `BOOL` | GET | Current debounced discrete state. |
| `Edge` | `E_SensorEdge` | GET | Enum on state transition (none, rising, falling or both are configurable). |

### Usage Pattern

```pascal
VAR
    _StartButton : I_DigitalSensor;
END_VAR

// React to edge — no timer logic needed in process code
IF _StartButton.Edge = RISING AND _StartButton.Value THEN
    // rising edge detected — operator pressed start
END_IF
```

---

## I_ComponentManager

The orchestration contract for component execution. Components register themselves with the Manager through this interface. Process logic calls `UpdateAll` through this interface. The concrete `FB_ComponentManager` is never referenced directly in application code.

**Implemented by:** `FB_ComponentManager`.

### Methods

| Method | Returns | Input | Description |
|---|---|---|---|
| `RegisterComponent` | `BOOL` | `Component : POINTER TO FB_ComponentBase` | Adds a component pointer to the execution array. |
| `UpdateAll` | `BOOL` | — | Executes the cyclic `Update()` method on all registered components via a polymorphic loop.loop. |
| `GetComponentByID` | `POINTER TO FB_ComponentBase` | `ID : UDINT` | Searches the registry and returns the interface reference for a specific component ID.

### Properties

| Properties | Type | Access | Description |
|---|---|---|---|
| `ComponentCount` | `POINTER TO UDINT` | GET | Returns the total number of components currently registered in the manager. |

### Usage Pattern

```pascal
// --- Registration (Initialization Phase) ---
GVL_Components.ComponentManager.RegisterComponent(Component := ADR(Pump_Hydraulic));
GVL_Components.ComponentManager.RegisterComponent(Component := ADR(Sensor_Pressure));

// --- Execution (Main Task) ---
GVL_Components.ComponentManager.UpdateAll();

// --- Retrieval (Diagnostic/Logic) ---
_tempComp := GVL_Components.ComponentManager.GetComponentByID(ID := 10);
IF _tempComp <> 0 THEN
    _isReady := _tempComp.Ready;
END_IF
```

---

## I_AlarmManager

The alarm routing contract. Components hold a reference to `I_AlarmManager` — never to `FB_AlarmManager` directly. This makes the alarm backend swappable: a mock implementation can replace the real manager in test environments without touching any component code.

**Implemented by:** `FB_AlarmManager`.

### Methods

| Method | Returns | Input | Description |
|---|---|---|---|
| `Update` | `BOOL` | — | Main cyclic executive. Polls components for new alarms and consolidates them into the global buffer. |
| `AcknowledgeAlarm` | `BOOL` |  `AlarmID : UDINT` |  Acknowledges a specific alarm by its unique ID. |
| `AcknowledgeAll` | `BOOL` | — | Sets `Acknowledge = TRUE` and records `AckTimestamp` on all currently active alarms. |
| `FilterByComponent` | `UDINT` | `ComponentID : UDINT` | Returns the count of active alarms for a specific component. |
| `Clear` | `BOOL` | — | Fully resets the alarm manager and wipes the global buffer. |
| `ClearAlarm` | `BOOL` | `AlarmID : UDINT` | Forcefully removes a specific alarm entry from the manager's list. |

### Properties

| Properties | Type | Access | Description |
|---|---|---|---|
| `ActiveAlarms` | `ARRAY[1..GVL_Config.MAX_ALARMS] OF ST_Alarm` | GET | Returns the global consolidated array of all active alarms in the system. |
| `AcknowledgeAll` | `UDINT` | GET | Returns the total number of alarms currently stored in the global buffer. |

### Usage Pattern

```pascal
// Cyclic update in MAIN
GVL_Components.AlarmManager.Update();

// Operator acknowledgement from HMI
IF HMI_AcknowledgeAll THEN
    GVL_Components.AlarmManager.AcknowledgeAll();
END_IF
```

---


