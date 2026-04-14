# TwinFrame · Technical Architecture

TwinFrame is built on four Software Engineering principles applied consistently across the entire codebase:

**Interface Segregation.** No component implements behavior it does not own. Nine focused interfaces replace a single monolithic contract. Each interface owns one responsibility and nothing else.

**Template Method Pattern.** `FB_ActuatorBase` defines the skeleton of the device control algorithm. Subclasses override specific steps (`DoStart`, `DoStop`, `DoUpdate`) without modifying the structure. Safety, state transitions, and fault propagation are never duplicated.

**Composition over Inheritance for cells.** Machine cells (`FB_PumpUnit`) do not inherit from their components — they own them. This follows the ISA-88 Equipment Module model and keeps cell logic decoupled from component implementation.

**Deterministic memory.** No dynamic allocation anywhere in the framework. All collections are fixed-size arrays. The alarm engine uses a static circular buffer with bounded worst-case footprint.

---

## Interface Layer

The interface layer is the contractual foundation of the framework. Process logic depends exclusively on these interfaces — never on concrete implementations. This is what makes HAL switching, polymorphic execution, and future unit testing structurally possible.

### I_Component

The primary lifecycle interface. Every object in the system implements this. It is the densest interface in the framework — intentionally, because lifecycle management is the one responsibility that every component shares without exception.

```pascal
INTERFACE I_Component

    // Lifecycle methods
    METHOD Init             : BOOL
    METHOD Update           : BOOL
    METHOD Reset            : BOOL
    METHOD Enable           : BOOL
    METHOD Disable          : BOOL
    METHOD Abort            : BOOL
    METHOD AcknowledgeAlarm : BOOL

    // Operational state
    PROPERTY State            : E_ComponentState             // GET
    PROPERTY Mode             : E_ExecutionMode              // GET
    PROPERTY Ready            : BOOL                         // GET
    PROPERTY Busy             : BOOL                         // GET
    PROPERTY ID               : UDINT                        // GET & SET

    // Alarm telemetry
    PROPERTY HasAlarm         : BOOL                         // GET
    PROPERTY ActiveAlarmCount : UDINT                        // GET
    PROPERTY ActiveAlarms     : ARRAY[1..N] OF ST_Alarm      // GET

END_INTERFACE
```

### I_Commandable

Decouples external command dispatch from lifecycle logic. Allows a supervisory layer or HMI to send typed commands without calling lifecycle methods directly.

```pascal
INTERFACE I_Commandable
    METHOD SendCommand : BOOL   // VAR_INPUT Cmd : ST_Command
END_INTERFACE
```

### I_StartStop

Explicit flow control for actuators and functional units. Keeps start/stop semantics separate from the full lifecycle contract of `I_Component`.

```pascal
INTERFACE I_StartStop
    METHOD Start     : BOOL
    METHOD Stop      : BOOL
    PROPERTY Running : BOOL  // GET
END_INTERFACE
```

### I_ErrorChecker

Isolates structured validation logic. A component that needs to validate its own state before acting implements this interface separately from its lifecycle contract.

```pascal
INTERFACE I_ErrorChecker
    METHOD CheckErrors : BOOL
END_INTERFACE
```

### Sensor Interfaces

`I_SensorBase` defines validity and unit metadata common to all sensors. `I_AnalogSensor` extends it with continuous value access and runtime parameter configuration. `I_DigitalSensor` extends it with discrete state and edge detection.

```pascal
INTERFACE I_SensorBase
    PROPERTY Valid : BOOL    // GET — signal quality flag
    PROPERTY Units : STRING  // GET & SET — engineering unit label

INTERFACE I_AnalogSensor EXTENDS I_SensorBase
    PROPERTY Value       : REAL   // GET — scaled engineering-unit value
    METHOD SetParameters : BOOL   // Runtime threshold configuration

INTERFACE I_DigitalSensor EXTENDS I_SensorBase
    PROPERTY State : BOOL         // GET — current debounced discrete state
    PROPERTY Edge  : BOOL         // GET — active for one cycle on configured transition
```

### Orchestration Interfaces

`I_ComponentManager` and `I_AlarmManager` keep both managers replaceable. Components register with and report alarms to interfaces — never to concrete implementations.

```pascal
INTERFACE I_ComponentManager
    METHOD Register          : BOOL
    METHOD UpdateAll         : BOOL
    METHOD GetComponentByID  : POINTER TO FB_ComponentBase
    PROPERTY ComponentCount  : UDINT

INTERFACE I_AlarmManager
    METHOD Update            : BOOL
    METHOD AcknowledgeAlarm  : BOOL
    METHOD AcknowledgeAll    : BOOL
    METHOD AlarmsByComponent : UDINT
    METHOD Clear             : BOOL
    METHOD ClearAlarm        : BOOL
    PROPERTY ActiveAlarms    : ARRAY[1..GVL_Config.MAX_ALARMS] OF ST_Alarm
    PROPERTY AlarmCount      : UDINT
```

---

## Data Model — Enums

All enums use `{attribute 'qualified_only'}` and `{attribute 'strict'}` to enforce explicit namespace qualification and prevent implicit type coercion — critical for safety in real-time environments.

### E_ComponentState — PackML-Aligned State Machine

26 states covering the full PackML operational lifecycle, plus maintenance, manual, and a custom extension base:

```pascal
{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_ComponentState :
(
    STATE_UNINITIALIZED := 0,    // Component created, not yet initialized
    STATE_INITIALIZING  := 10,   // Initialization in progress
    STATE_IDLE          := 20,   // Ready, awaiting commands
    STATE_STARTING      := 30,   // Pre-run preparation
    STATE_RUNNING       := 40,   // Normal operation
    STATE_EXECUTE       := 41,   // Executing specific task within RUN
    STATE_STOPPING      := 50,   // Controlled stop in progress
    STATE_STOPPED       := 60,   // Inactive
    STATE_PAUSING       := 70,   // Pause initiated
    STATE_PAUSED        := 80,   // Paused
    STATE_RESUMING      := 90,   // Resuming from pause
    STATE_ABORTING      := 100,  // Emergency abort in progress
    STATE_ABORTED       := 110,  // Aborted — requires reset
    STATE_HOLDING       := 120,  // Entering HOLD
    STATE_HELD          := 130,  // HOLD active
    STATE_UNHOLDING     := 140,  // Exiting HOLD
    STATE_SUSPENDING    := 150,  // Suspension initiated
    STATE_SUSPENDED     := 160,  // Suspension active
    STATE_UNSUSPENDING  := 170,  // Resuming from suspension
    STATE_COMPLETING    := 180,  // Finalizing before complete
    STATE_COMPLETE      := 190,  // Task completed successfully
    STATE_ERROR         := 200,  // Error detected
    STATE_FAULT         := 201,  // Critical fault — emergency stop
    STATE_MAINTENANCE   := 250,  // Maintenance mode
    STATE_MANUAL        := 251,  // Manual operation
    STATE_CUSTOM_BASE   := 1000  // Base for project-specific extensions
);
END_TYPE
```

The gap between `STATE_FAULT` (201) and `STATE_MAINTENANCE` (250) is intentional — it leaves room for intermediate error states in project-specific extensions without colliding with the standard model.

### E_CommandType — Full PackML Command Set

18 commands covering initialization, flow control, the complete Hold/Suspend/Complete cycle, and operational utilities:

```pascal
{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_CommandType :
(
    CMD_NONE        := 0,    // No command
    CMD_INIT        := 1,    // Initialize component
    CMD_START       := 2,    // Start operation
    CMD_STOP        := 3,    // Stop operation
    CMD_PAUSE       := 4,    // Pause operation
    CMD_RESUME      := 5,    // Resume operation
    CMD_ABORT       := 6,    // Abort operation completely
    CMD_RESET       := 7,    // Reset errors or states
    CMD_ACKNOWLEDGE := 8,    // Acknowledge alarms or events
    CMD_ENABLE      := 9,    // Enable component
    CMD_DISABLE     := 10,   // Disable component
    CMD_HOLD        := 11,   // Enter hold state
    CMD_UNHOLD      := 12,   // Exit hold state
    CMD_SUSPEND     := 13,   // Temporarily suspend operation
    CMD_UNSUSPEND   := 14,   // Resume suspended operation
    CMD_COMPLETE    := 15,   // Mark operation as complete
    CMD_CLEAR       := 16,   // Clear alarms or flags
    CMD_RESTART     := 17    // Restart component or operation
);
END_TYPE
```

### E_ExecutionMode — Operational Context

Separates *what mode the system is in* from *what state the component is in*. A component can be in `STATE_RUNNING` while the system is in `MODE_SIMULATION` or `MODE_TEST` — the state machine behaviour remains identical, but I/O routing changes at the HAL level.

```pascal
{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_ExecutionMode :
(
    MODE_MANUAL      := 0,   // Operator manual control
    MODE_SEMI_AUTO   := 1,   // Mixed manual/automatic
    MODE_AUTOMATIC   := 2,   // Fully automatic
    MODE_SIMULATION  := 3,   // Simulation — no real hardware affected
    MODE_MAINTENANCE := 4,   // Maintenance and adjustment
    MODE_TEST        := 5,   // Logic verification
    MODE_DRY_RUN     := 6    // Process dry run — outputs suppressed
);
END_TYPE
```

### E_AlarmSeverity — 8-Level Severity Model

Non-linear severity scale: gaps between levels allow future intermediate severities without restructuring conditional logic that compares severity values numerically.

```pascal
{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_AlarmSeverity :
(
    SEVERITY_NONE     := 0,    // No alarm condition
    SEVERITY_TRACE    := 1,    // Internal trace — very low level
    SEVERITY_DEBUG    := 2,    // Debug information for engineers
    SEVERITY_INFO     := 10,   // General status — normal operation
    SEVERITY_WARNING  := 20,   // Attention required, operation continues
    SEVERITY_ERROR    := 30,   // Error — partial operation affected
    SEVERITY_CRITICAL := 40,   // Critical — immediate intervention required
    SEVERITY_FATAL    := 50    // Fatal — operation stopped, restart required
);
END_TYPE
```

### E_EventType — Structured Event Categories

Separates alarm events from state changes, commands, and diagnostic events for structured logging and OPC UA EventNotifier emission:

```pascal
{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_EventType :
(
    EVENT_NONE          := 0,
    EVENT_STATE_CHANGE  := 1,
    EVENT_ALARM_RAISED  := 2,
    EVENT_ALARM_CLEARED := 3,
    EVENT_ALARM_ACK     := 4,
    EVENT_CMD_RECEIVED  := 5,
    EVENT_FAULT         := 6,
    EVENT_DIAGNOSTIC    := 7,
    EVENT_CONFIG_CHANGE := 8
);
END_TYPE
```

---

## Data Model — Structures

### ST_Alarm

The atomic unit of the alarm system. Carries everything needed for traceability without external lookups:

```pascal
TYPE ST_Alarm :
STRUCT
    ID           : UDINT;            // Internal ID assigned by FB_AlarmQueue
    ComponentID  : UDINT;            // Source component identifier
    AlarmCode    : UDINT;            // Unique alarm type code per component class
    Severity     : E_AlarmSeverity;  // Operational impact level
    IsFault      : BOOL;             // TRUE — triggers STATE_FAULT in owning component
    Active       : BOOL;             // Alarm condition currently present
    Acknowledge  : BOOL;             // Operator has acknowledged this alarm
    Timestamp    : DATE_AND_TIME;    // Fault trigger time
    AckTimestamp : DATE_AND_TIME;    // Acknowledgement time
END_STRUCT
END_TYPE
```

The dual timestamp (`Timestamp` / `AckTimestamp`) enables response time analysis — the interval between fault detection and operator acknowledgement is directly computable from the structure with no additional logging infrastructure.

### ST_Component and ST_ComponentStatus

The runtime identity and health snapshot of any component. Designed for direct OPC UA node exposure:

```pascal
TYPE ST_Component :
STRUCT
    ID     : UDINT;
    Status : ST_ComponentStatus;
END_STRUCT
END_TYPE

TYPE ST_ComponentStatus :
STRUCT
    State       : E_ComponentState;  // Current PackML state
    Mode        : E_ExecutionMode;   // Current execution mode
    Ready       : BOOL;              // TRUE when component accepts commands
    Busy        : BOOL;              // TRUE when actively executing a task
    Error       : BOOL;              // TRUE when a fault condition is present
    LastCommand : E_CommandType;     // Last command received
    Timestamp   : DATE_AND_TIME;     // Last state or status update
END_STRUCT
END_TYPE
```

### ST_Diagnostic

Production-grade telemetry. Populated by `FB_ComponentBase` on every cycle without application-layer instrumentation:

```pascal
TYPE ST_Diagnostic :
STRUCT
    UpTime          : TIME;           // Total runtime since initialization
    ErrorCount      : UDINT;          // Total errors detected during operation
    LastErrorCode   : UDINT;          // Most recent error code
    CycleCount      : UDINT;          // Execution cycles since startup
    LastStateChange : DATE_AND_TIME;  // Timestamp of the last state transition
END_STRUCT
END_TYPE
```

### ST_Command

Commands are typed structures with a dispatch timestamp, enabling command history and audit logging:

```pascal
TYPE ST_Command :
STRUCT
    Cmd       : E_CommandType;  // Command type to be executed
    Execute   : BOOL;           // Execution trigger — rising edge initiates command
    Timestamp : DT;             // Timestamp when the command was issued
END_STRUCT
END_TYPE
```

---

## FB_ComponentBase — The Abstract Root

`FB_ComponentBase` is the foundation of the entire hierarchy. It is never instantiated directly — only inherited. Every sensor, actuator, and functional unit in the system carries this infrastructure automatically.

```pascal
FUNCTION_BLOCK ABSTRACT FB_ComponentBase IMPLEMENTS I_Component
VAR
    Info        : ST_Component;   // Component identity and status snapshot
    Initialized : BOOL := FALSE;  // First-cycle initialization guard
    AlarmQueue  : FB_AlarmQueue;  // Local circular alarm buffer
    _Diagnostic : ST_Diagnostic;  // Production telemetry
    _CycleTimer : TON;            // Internal cycle timer
END_VAR
```

| Method | Responsibility |
|---|---|
| `Init` | Parameter binding and first-cycle initialization |
| `Update` | Cyclic entry point — state machine tick and telemetry update |
| `Reset` | Fault recovery — transitions from `STATE_ABORTED` or `STATE_ERROR` to `STATE_IDLE` |
| `Enable` / `Disable` | Runtime activation control |
| `Abort` | Emergency stop — immediate transition to `STATE_ABORTING` |
| `SetAlarm` | Sole alarm generation path — populates and pushes to `AlarmQueue` |
| `ClearAlarm` | Deactivates a specific alarm by code |
| `AcknowledgeAlarm` | Marks active alarms on this instance as operator-acknowledged |

`SetAlarm` is the only path by which a component generates an alarm. There is no direct access to `AlarmQueue` from subclasses or application logic. This enforces consistent alarm formatting and prevents ID collisions across the component tree.

---

## FB_Watchdog — Independent Safety Timer

`FB_Watchdog` is a standalone utility block composed into `FB_ComponentBase`, not inherited. It can be instantiated independently wherever timeout detection is needed outside the component hierarchy.

```pascal
FUNCTION_BLOCK FB_Watchdog
VAR
    _Timer     : TON;
    _Active    : BOOL;
    _Timeout   : BOOL;
    _TimeLimit : TIME;
END_VAR
```

| Method | Responsibility |
|---|---|
| `Start` | Arms the timer with the configured timeout threshold |
| `Stop` | Disarms the timer and resets the timeout flag |
| `Update` | Advances the internal TON — must be called every cycle |

| Property | Description |
|---|---|
| `IsActive` | TRUE when the watchdog is armed and running |
| `IsTimeout` | TRUE when the monitored condition did not arrive within the threshold |

When `IsTimeout` fires, the owning component reads it in its `Update` cycle and calls `SetAlarm` with the appropriate severity. Composition rather than inheritance is used here so that a component can hold multiple independent `FB_Watchdog` instances — for example, one monitoring hardware feedback and another monitoring a communication heartbeat.

---

## FB_AlarmQueue — Deterministic Alarm Engine

`FB_AlarmQueue` implements a static alarm buffer for local component alarm storage. No dynamic allocation. Bounded worst-case memory footprint known at compile time.

```pascal
FUNCTION_BLOCK FB_AlarmQueue
VAR
    _Alarms : ARRAY[1..GVL_Config.MAX_ALARMS_PER_COMPONENT] OF ST_Alarm;
    _Count  : UDINT := 0;
END_VAR
```

`Push` is the critical path. It is called internally by `FB_ComponentBase.SetAlarm` and is the sole entry point into the alarm buffer. It enforces duplicate suppression — if an alarm with the same `AlarmCode` and `ComponentID` is already active, the push is rejected:

```pascal
METHOD Push : BOOL
VAR_INPUT
    Code        : UDINT;
    ComponentID : UDINT;
    Severity    : E_AlarmSeverity := E_AlarmSeverity.SEVERITY_WARNING;
END_VAR
VAR
    i : UDINT;
END_VAR

// Capacity guard
IF _Count >= GVL_Config.MAX_ALARMS_PER_COMPONENT THEN
    Push := FALSE;
    RETURN;
END_IF

// Duplicate suppression — reject if already active
FOR i := 1 TO _Count DO
    IF _Alarms[i].AlarmCode = Code AND
       _Alarms[i].ComponentID = ComponentID THEN
        Push := FALSE;
        RETURN;
    END_IF
END_FOR

// Insert new alarm record
_Count := _Count + 1;
_Alarms[_Count].ID          := _Count;
_Alarms[_Count].ComponentID := ComponentID;
_Alarms[_Count].AlarmCode   := Code;
_Alarms[_Count].Severity    := Severity;
_Alarms[_Count].IsFault     := Severity >= E_AlarmSeverity.SEVERITY_CRITICAL;
_Alarms[_Count].Active      := TRUE;
_Alarms[_Count].Acknowledge := FALSE;
_Alarms[_Count].Timestamp   := GVL_Components.CurrentDT;

Push := TRUE;
```

`IsFault` is derived automatically from severity at insertion time — any alarm at `SEVERITY_CRITICAL` or above sets `IsFault := TRUE`, which propagates upward through `I_Component.HasFault` to signal the application layer that an emergency response is required.

---

## Actuator Branch — Template Method Pattern

`FB_ActuatorBase` applies the Template Method pattern to device control. The base class defines the invariant algorithm skeleton — state transitions, safety interlocks, watchdog monitoring, and fault propagation. Subclasses implement device-specific behavior by overriding six protected hook methods without touching the state machine.

### Protected Hooks

| Method | Responsibility |
|---|---|
| `DoStart` | Activates physical outputs to start the device |
| `DoCheckStarted` | Returns TRUE when feedback confirms the device has reached its active state |
| `DoUpdate` | Cyclic logic executed on every cycle regardless of PackML state |
| `DoCheckStopped` | Returns TRUE when feedback confirms the device has reached a safe stop |
| `DoStop` | De-energizes physical outputs |
| `ProcessInputs` | Maps hardware inputs to internal status flags before state evaluation |

```pascal
FUNCTION_BLOCK ABSTRACT FB_ActuatorBase EXTENDS FB_ComponentBase
                                        IMPLEMENTS I_StartStop, I_Commandable
VAR
    Command           : ST_Command;        // Current active command
    LastCommand       : E_CommandType;     // Last executed command
    Watchdog          : FB_Watchdog;       // Integrated safety timer
    WatchdogStartTime : TIME := T#10S;     // Timeout threshold for starting
    WatchdogStopTime  : TIME := T#10S;     // Timeout threshold for stopping
END_VAR
```

### Template Method: Update

`Update` is the invariant skeleton. It coordinates pre-processing, safety checks, and state machine execution in a fixed order that subclasses cannot modify:

```pascal
METHOD Update : BOOL

// 1. Pre-processing
Watchdog.Update();
ProcessInputs();  // Hook: map hardware inputs to internal flags
DoUpdate();       // Hook: general cyclic logic

// 2. Fault priority — immediate abort on active fault alarm
IF AlarmQueue.HasFault THEN
    Info.Status.State := E_ComponentState.STATE_ABORTED;
    Info.Status.Busy  := FALSE;
    Info.Status.Ready := FALSE;
    Info.Status.Error := TRUE;
    SUPER^.Update();
    Update := TRUE;
    RETURN;
END_IF

// 3. Watchdog timeout — transition to aborted
IF Watchdog.IsTimeout THEN
    SetAlarm(Code     := GVL_AlarmCodes.ALARM_TIMEOUT,
             Severity := E_AlarmSeverity.SEVERITY_CRITICAL);
    Info.Status.State := E_ComponentState.STATE_ABORTED;
    Info.Status.Busy  := FALSE;
    Info.Status.Ready := FALSE;
    Watchdog.Stop();
    SUPER^.Update();
    Update := TRUE;
    RETURN;
END_IF

// 4. PackML state machine
CASE Info.Status.State OF

    E_ComponentState.STATE_IDLE:
        Info.Status.Ready := TRUE;
        Info.Status.Busy  := FALSE;

    E_ComponentState.STATE_STARTING:
        Info.Status.Busy  := TRUE;
        Info.Status.Ready := FALSE;
        DoStart();                    // Hook: activate physical output
        IF DoCheckStarted() THEN      // Hook: confirm device running
            Watchdog.Stop();
            ClearAlarm(Code := GVL_AlarmCodes.ALARM_TIMEOUT);
            Info.Status.State := E_ComponentState.STATE_RUNNING;
        END_IF

    E_ComponentState.STATE_RUNNING:
        Info.Status.Busy  := TRUE;
        Info.Status.Ready := FALSE;

    E_ComponentState.STATE_STOPPING:
        Info.Status.Busy  := TRUE;
        Info.Status.Ready := FALSE;
        DoStop();                     // Hook: de-energize output
        IF DoCheckStopped() THEN      // Hook: confirm device stopped
            Watchdog.Stop();
            ClearAlarm(Code := GVL_AlarmCodes.ALARM_TIMEOUT);
            Info.Status.State := E_ComponentState.STATE_STOPPED;
            Info.Status.Busy  := FALSE;
            Info.Status.Ready := TRUE;
        END_IF

    E_ComponentState.STATE_STOPPED:
        Info.Status.Busy  := FALSE;
        Info.Status.Ready := TRUE;

    E_ComponentState.STATE_ABORTED:
        Info.Status.Busy  := FALSE;
        Info.Status.Ready := FALSE;

END_CASE

SUPER^.Update();  // Base class: uptime, cycle count, telemetry
Update := TRUE;
```

A concrete block such as `FB_Pump` inherits all safety and state logic. The developer implements only the six hook methods. Subclasses cannot manipulate state transitions directly or bypass the watchdog — all transitions are enforced by the base template. I/O binding is handled via dedicated structures (e.g., `ST_CellPumpIO`), so swapping field devices requires no changes to control logic.

---

## Sensor Branch — Typed Signal Hierarchy

The sensor branch implements a clean specialization hierarchy. Each level adds only what is specific to its signal domain.

### FB_SensorBase

All sensors inherit from this abstract base, which provides the common signal quality flag and engineering unit metadata:

```pascal
FUNCTION_BLOCK ABSTRACT FB_SensorBase EXTENDS FB_ComponentBase
                                      IMPLEMENTS I_SensorBase, I_ErrorChecker
VAR
    Sensor : ST_SensorBase;
END_VAR

TYPE ST_SensorBase :
STRUCT
    Valid : BOOL;        // Signal quality — FALSE on wire-break or out-of-range
    Units : STRING[10];  // Engineering unit label (e.g. 'bar', '°C', 'L/min')
END_STRUCT
END_TYPE
```

### FB_AnalogSensor

Manages continuous signal acquisition through a processing pipeline: raw-to-engineering-unit scaling, optional deadband filtering, and four-level threshold monitoring.

```pascal
FUNCTION_BLOCK FB_AnalogSensor EXTENDS FB_SensorBase IMPLEMENTS I_AnalogSensor
VAR
    Analog : ST_AnalogSensor;
END_VAR

TYPE ST_AnalogSensor :
STRUCT
    Value : REAL;  // Scaled engineering-unit value
    Lo    : REAL;  // Low warning threshold
    LoLo  : REAL;  // Low-Low critical threshold
    Hi    : REAL;  // High warning threshold
    HiHi  : REAL;  // High-High critical threshold
END_STRUCT
END_TYPE
```

Threshold violations generate typed alarms via `E_AnalogAlarm`, a dedicated enum that keeps analog fault codes isolated from actuator or system-level alarm namespaces:

```pascal
{attribute 'qualified_only'}
TYPE E_AnalogAlarm :
(
    NONE := 0,
    LOLO := 1,
    LO   := 2,
    HI   := 3,
    HIHI := 4
);
END_TYPE
```

`SetParameters` (from `I_AnalogSensor`) allows threshold reconfiguration at runtime without reinitializing the component — used by `FB_PumpUnit.Configure` during the startup state machine.

### FB_DigitalSensor

Manages discrete signal acquisition with built-in debounce and configurable edge detection. Instead of external `R_TRIG` or `F_TRIG` instances, the application layer configures the desired edge behavior via `E_SensorEdge`. The sensor owns its transition logic internally.

```pascal
FUNCTION_BLOCK FB_DigitalSensor EXTENDS FB_SensorBase IMPLEMENTS I_DigitalSensor
VAR
    Digital : ST_DigitalSensor;
END_VAR

TYPE ST_DigitalSensor :
STRUCT
    Value      : BOOL;          // Current debounced state
    LastValue  : BOOL;          // Previous cycle state for edge computation
    Edge       : BOOL;          // TRUE for one cycle on the configured transition
    SensorEdge : E_SensorEdge;  // Edge detection configuration
END_STRUCT
END_TYPE

{attribute 'qualified_only'}
{attribute 'strict'}
TYPE E_SensorEdge :
(
    NONE    := 0,  // No edge detection
    RISING  := 1,  // Detect 0 → 1 transition
    FALLING := 2,  // Detect 1 → 0 transition
    BOTH    := 3   // Detect any transition
);
END_TYPE
```

---

## FB_ComponentManager — Polymorphic Orchestration

`FB_ComponentManager` is the runtime execution engine. It holds a fixed-size array of pointers to `FB_ComponentBase` and executes every registered component through a single deterministic cyclic loop.

```pascal
FUNCTION_BLOCK FB_ComponentManager IMPLEMENTS I_ComponentManager
VAR
    Components        : ARRAY[1..GVL_Config.MAX_COMPONENTS] OF POINTER TO FB_ComponentBase;
    ComponentCountVar : UDINT := 0;
END_VAR
```

`RegisterComponent` handles polymorphic registration. By accepting a pointer to the abstract base, it accepts any subclass — pump, sensor, valve, or any future type — through the same method:

```pascal
METHOD RegisterComponent : BOOL
VAR_INPUT
    Component : POINTER TO FB_ComponentBase;
END_VAR
VAR
    i : UDINT;
END_VAR

// Guard: null pointer
IF Component = 0 THEN
    RegisterComponent := FALSE;
    RETURN;
END_IF

// Guard: capacity
IF ComponentCountVar >= GVL_Config.MAX_COMPONENTS THEN
    RegisterComponent := FALSE;
    RETURN;
END_IF

// Guard: duplicate registration
FOR i := 1 TO ComponentCountVar DO
    IF Components[i] = Component THEN
        RegisterComponent := FALSE;
        RETURN;
    END_IF
END_FOR

// Register and auto-assign ID
ComponentCountVar := ComponentCountVar + 1;
Components[ComponentCountVar] := Component;
Component^.ID := ComponentCountVar;

RegisterComponent := TRUE;
```

Auto-assigning `Component^.ID` at registration ensures the ID inside the component's `ST_ComponentStatus` always matches the Manager's index. This is what allows `FB_AlarmManager` to identify the source of any fault without additional mapping.

`UpdateAll` executes the polymorphic loop:

```pascal
METHOD UpdateAll : BOOL
VAR
    i : UDINT;
END_VAR

IF ComponentCountVar = 0 THEN
    UpdateAll := TRUE;
    RETURN;
END_IF

FOR i := 1 TO ComponentCountVar DO
    IF Components[i] <> 0 THEN
        Components[i]^.Update();  // Polymorphic dispatch via pointer
    END_IF
END_FOR

UpdateAll := TRUE;
```

The loop iterates only up to `ComponentCountVar`, not the full array capacity. Task cycle time remains constant regardless of how large `MAX_COMPONENTS` is configured. In IEC 61131-3, references cannot be stored in arrays — pointer-based storage is the correct and only mechanism for heterogeneous runtime collections. The null check on every iteration is a mandatory safety guard.

---

## FB_AlarmManager — Centralized Event Routing

`FB_AlarmManager` is the system-level alarm aggregator. It does not own primary alarm storage — each component owns its `FB_AlarmQueue`. The Manager polls all registered components, consolidates active alarms into a system-wide buffer, and provides operator-level acknowledgement and filtering.

```pascal
FUNCTION_BLOCK FB_AlarmManager IMPLEMENTS I_AlarmManager
VAR
    _Alarms      : ARRAY[1..GVL_Config.MAX_ALARMS] OF ST_Alarm;
    _AlarmCount  : UDINT := 0;
    _WriteIndex  : UDINT := 1;
END_VAR
```

`Update` performs two operations on every cycle — registering new alarms and deactivating resolved ones:

```pascal
METHOD Update : BOOL
VAR
    i, j, k      : UDINT;
    _comp        : POINTER TO FB_ComponentBase;
    _compAlarms  : ARRAY[1..50] OF ST_Alarm;
    _compCount   : UDINT;
    _exists      : BOOL;
    _stillActive : BOOL;
END_VAR

FOR i := 1 TO GVL_Components.ComponentManager.ComponentCount DO
    _comp := GVL_Components.ComponentManager.GetComponentByID(i);

    IF _comp <> 0 THEN
        _compAlarms := _comp^.ActiveAlarms;
        _compCount  := _comp^.ActiveAlarmCount;

        // Part 1: Register new alarms not yet in the system buffer
        FOR k := 1 TO _compCount DO
            _exists := FALSE;
            FOR j := 1 TO GVL_Config.MAX_ALARMS DO
                IF _Alarms[j].AlarmCode   = _compAlarms[k].AlarmCode AND
                   _Alarms[j].ComponentID = _compAlarms[k].ComponentID AND
                   _Alarms[j].Active THEN
                    _exists := TRUE;
                END_IF
            END_FOR

            IF NOT _exists THEN
                // Circular buffer: write at current index, then advance with wrap-around
                _Alarms[_WriteIndex]    := _compAlarms[k];
                _Alarms[_WriteIndex].ID := _WriteIndex;
                IF _WriteIndex >= GVL_Config.MAX_ALARMS THEN
                    _WriteIndex := 1;
                ELSE
                    _WriteIndex := _WriteIndex + 1;
                END_IF
                IF _AlarmCount < GVL_Config.MAX_ALARMS THEN
                    _AlarmCount := _AlarmCount + 1;
                END_IF
            END_IF
        END_FOR

        // Part 2: Deactivate alarms no longer present in the source component
        FOR j := 1 TO GVL_Config.MAX_ALARMS DO
            IF _Alarms[j].ComponentID = _comp^.ID AND _Alarms[j].Active THEN
                _stillActive := FALSE;
                FOR k := 1 TO _compCount DO
                    IF _compAlarms[k].AlarmCode = _Alarms[j].AlarmCode THEN
                        _stillActive := TRUE;
                    END_IF
                END_FOR
                IF NOT _stillActive THEN
                    _Alarms[j].Active := FALSE;
                END_IF
            END_IF
        END_FOR

    END_IF
END_FOR

Update := TRUE;
```

The bidirectional update — registering new alarms and deactivating resolved ones in the same cycle — ensures the system alarm list always reflects the true current state of every component. Since each component owns its `FB_AlarmQueue`, there is no single point of failure and no memory bottleneck. `FilterByComponent` allows HMI screens to display per-device alarm history. `AcknowledgeAll` iterates all active alarms and sets the acknowledgement flag and timestamp in a single operator action.

---

## Machine Cells — ISA-88 Functional Units

Machine cells are the application layer. They follow composition over inheritance — they do not extend the component hierarchy, they own instances of it. `FB_PumpUnit` is the reference ISA-88 Equipment Module, aggregating a pump actuator, three analog sensors (pressure, flow, temperature), and three digital sensor inputs (start, stop, reset buttons).

`Configure` is called once during the startup state machine. It initializes all internal components and distributes threshold parameters to each analog sensor via `SetParameters`:

```pascal
METHOD Configure : BOOL
VAR_INPUT
    PresLo : LREAL;  PresLoLo : LREAL;  PresHi : LREAL;  PresHiHi : LREAL;
    FlowLo : LREAL;  FlowLoLo : LREAL;  FlowHi : LREAL;  FlowHiHi : LREAL;
    TempLo : LREAL;  TempLoLo : LREAL;  TempHi : LREAL;  TempHiHi : LREAL;
END_VAR

Temp.Init();
Temp.SetParameters(lo := TempLo, lolo := TempLoLo, hi := TempHi, hihi := TempHiHi, units := '°C');
Flow.Init();
Flow.SetParameters(lo := FlowLo, lolo := FlowLoLo, hi := FlowHi, hihi := FlowHiHi, units := 'rpm');
Pres.Init();
Pres.SetParameters(lo := PresLo, lolo := PresLoLo, hi := PresHi, hihi := PresHiHi, units := 'mm/s');

bStart.Init(); bStart.Edge := E_SensorEdge.RISING;
bStop.Init();  bStop.Edge  := E_SensorEdge.RISING;
bReset.Init(); bReset.Edge := E_SensorEdge.RISING;
```

`SetDigitalInputs` is the only point in the system where raw fieldbus values are touched. By distributing them to sensor objects at the start of each cycle, all cell logic above this method remains completely hardware-independent:

```pascal
METHOD SetDigitalInputs : BOOL
VAR_INPUT
    StartVal : BOOL;
    StopVal  : BOOL;
    ResetVal : BOOL;
END_VAR

bStart.Value := StartVal;
bStop.Value  := StopVal;
bReset.Value := ResetVal;
```

---

## Hardware Abstraction Layer

The HAL is implemented at the concrete component level via `GVL_Components.SimulationMode`.
The pattern differs between actuators and sensors:

**Actuators — last-mile output protection:**

```pascal
METHOD PROTECTED DoStart : BOOL
IF NOT GVL_Components.SimulationMode THEN
    IO.Q_Start := TRUE;  // Physical write only in real mode
END_IF
DoStart := TRUE;  // Logical execution continues regardless
```

**Sensors — input substitution:**

```pascal
IF GVL_Components.SimulationMode THEN
    _RawValue := _SimValue;         // Forced simulation input
ELSE
    _RawValue := _IO.PhysicalInput; // Real hardware input
END_IF
```

A single GVL assignment switches the entire system — no individual component
instances need to be touched:

```pascal
GVL_Components.SimulationMode := TRUE;   // Development mode — logic only
GVL_Components.SimulationMode := FALSE;  // Full hardware mode
```

This HAL validates control logic, state machine behaviour, and alarm propagation.
It does not model physical dynamics, fieldbus delays, or signal noise.

---

## Design Decisions & Trade-offs

**`{attribute 'strict'}` on all enums.** Prevents implicit integer-to-enum assignment, catching type errors at compile time rather than runtime. The minor inconvenience of explicit casting is worth the safety guarantee in a real-time environment.

**Non-linear severity values in `E_AlarmSeverity`.** Values 1, 2, 10, 20, 30, 40, 50 rather than 0–7. This leaves gaps for future intermediate severities without breaking existing comparisons — a `>= SEVERITY_CRITICAL` check remains correct if a `SEVERITY_CRITICAL_PLUS := 45` is added later.

**`STATE_CUSTOM_BASE := 1000` in `E_ComponentState`.** Project-specific extensions start well above the standard PackML range. Custom states can be added per-project without collision risk and without modifying the base enum.

**Dual timestamp on `ST_Alarm`.** `Timestamp` records when the fault occurred. `AckTimestamp` records when the operator acknowledged it. Response time is computable directly from the structure with no additional infrastructure.

**`I_AlarmManager` on components, not `FB_AlarmManager`.** Components hold a reference to the interface, not the concrete manager. The alarm backend is fully swappable — a mock implementation replaces the real manager in test environments without touching any component code.

**Composition for `FB_Watchdog`.** Composed into `FB_ComponentBase` rather than inherited, allowing multiple independent watchdog instances per component and keeping `FB_Watchdog` usable outside the component hierarchy.

**`DoCheckStarted` and `DoCheckStopped` as separate hooks.** Separating the action (`DoStart`) from the confirmation (`DoCheckStarted`) allows the base class to manage transition timing and watchdog interaction independently of the device-specific feedback logic. A pump and a valve can have completely different confirmation criteria without affecting the state machine structure.

---
