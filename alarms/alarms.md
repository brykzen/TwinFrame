# TwinFrame · Alarm Engine

Alarm management in most PLC projects is an afterthought. Alarms are wired manually — a fault bit here, a message string there, a SCADA mapping somewhere else. The result is inconsistent coverage, forgotten fault paths, and alarm systems that require as much maintenance as the machine itself.

TwinFrame takes the opposite approach: **alarms are structural, not wired**.

Every component in the system generates, stores, and propagates alarms through the same mechanism, inherited automatically from `FB_ComponentBase`. There is no manual alarm wiring in the application layer. There are no forgotten fault paths. A machine with fifty components has a fully functional, centralized alarm system with zero alarm-wiring code outside the framework.

Three principles govern the design:

**Deterministic memory.** All alarm storage uses fixed-size static arrays. No dynamic allocation anywhere in the alarm system. Worst-case memory footprint is known at compile time.

**Traceability by default.** Every alarm carries its source component ID, a unique alarm code, a severity level, an active/acknowledged status, and dual timestamps — fault trigger time and acknowledgement time. Traceability requires no additional infrastructure.

**Decoupled propagation.** Components push alarms through the `I_AlarmManager` interface, never to `FB_AlarmManager` directly. The alarm backend is swappable — a mock implementation replaces the real manager in test environments without touching any component code.

---

## E_AlarmSeverity — The Severity Model

Eight severity levels on a non-linear scale. The gaps between values are intentional — they allow future intermediate severities to be inserted without restructuring any conditional logic that compares severity values numerically.

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

**Severity thresholds in practice:**

| Range | Behavior |
|---|---|
| `SEVERITY_NONE` to `SEVERITY_INFO` | Informational — logged, not alarmed. Does not affect component state. |
| `SEVERITY_WARNING` | Logged and surfaced to operator. Component continues operating. |
| `SEVERITY_ERROR` | Component transitions to `STATE_ERROR`. Partial operation may continue. |
| `SEVERITY_CRITICAL` | Component transitions to `STATE_FAULT`. Immediate intervention required. `IsFault = TRUE`. |
| `SEVERITY_FATAL` | Component transitions to `STATE_FAULT`. Full stop. `IsFault = TRUE`. Restart required. |

The `{attribute 'strict'}` pragma prevents implicit integer assignment — severity levels cannot be set by raw integer in process logic, only by explicit enum member. This eliminates a class of silent errors common in systems that use integer severity codes.

---

## E_EventType — Structured Event Categories

Separates alarm events from state changes, command receipts, and diagnostic events. Used by the alarm engine and any structured logging or OPC UA EventNotifier implementation:

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

`EVENT_STATE_CHANGE` and `EVENT_ALARM_RAISED` are the two highest-frequency events in normal operation. Separating them allows a future logging system to filter by event type — an operator console shows only `EVENT_ALARM_RAISED` and `EVENT_ALARM_CLEARED`, while an engineering diagnostic view shows all event types including `EVENT_DIAGNOSTIC` and `EVENT_STATE_CHANGE`.

---

## ST_Alarm — The Atomic Alarm Unit

Every alarm in the system is an instance of `ST_Alarm`. The structure carries everything needed for traceability without external lookups:

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

**Key field decisions:**

`IsFault` is separate from `Severity`. A `SEVERITY_CRITICAL` alarm may or may not trigger a fault state depending on the component's safety requirements. `IsFault = TRUE` is the explicit flag that drives the state machine to `STATE_FAULT` — severity alone does not.

The dual timestamp (`Timestamp` / `AckTimestamp`) enables **response time analysis** — the interval between fault detection and operator acknowledgement is directly computable from the structure with no additional logging infrastructure.

`AlarmCode` is unique per component class, not system-wide. `FB_AnalogSensor` uses `E_AnalogAlarm` codes. `FB_ActuatorBase` uses actuator-specific codes. This namespace separation prevents code collisions across the component tree and allows per-class alarm dictionaries.

---

## FB_AlarmQueue

`FB_AlarmQueue` manages the transient state of alarms within a single component. Unlike the system-wide manager, this queue does not store history; it maintains a dynamic list of active conditions. When an alarm is cleared, the list collapses to ensure the active alarms always occupy the first contiguous slots of the array.

```pascal
FUNCTION_BLOCK FB_AlarmQueue
VAR
    _Alarms : ARRAY[1..MAX_ALARMS_PER_COMPONENT] OF ST_Alarm;
    _Count  : UDINT := 0;
END_VAR
```

### Push — Fault Detection

Called by `FB_ComponentBase.SetAlarm`. It implements a duplicate check to ensure the same alarm code isn't registered multiple times. If the alarm is new, it is appended to the end of the active list.

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

// Check Capacity
IF _Count >= GVL_Config.MAX_ALARMS_PER_COMPONENT THEN
    Push := FALSE;
    RETURN;
END_IF

// Avoid Duplicates
FOR i := 1 TO _Count DO
    IF _Alarms[i].AlarmCode = Code AND 
       _Alarms[i].ComponentID = ComponentID THEN
        Push := FALSE;
        RETURN;
    END_IF
END_FOR

// Append to list
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

### Acknowledge

Records that an operator has seen the alarm. The alarm remains in the list until the physical condition is resolved (via `Clear`).

```pascal
METHOD Acknowledge : BOOL
VAR_INPUT
    Code        : UDINT;
    ComponentID : UDINT;
END_VAR
VAR
    i : UDINT;
END_VAR

FOR i := 1 TO _Count DO
    IF _Alarms[i].AlarmCode = Code AND 
       _Alarms[i].ComponentID = ComponentID THEN
        _Alarms[i].Acknowledge := TRUE;
		_Alarms[i].AckTimestamp   := GVL_Components.CurrentDT;
        Acknowledge := TRUE;
        RETURN;
    END_IF
END_FOR
Acknowledge := FALSE;
```
### Clear

When a fault condition disappears, Clear is called. This method removes the entry and shifts all subsequent alarms one position up. This ensures the `_Alarms` array never has "holes," allowing the `AlarmManager` to poll the component efficiently.

```pascal
METHOD Clear: BOOL
VAR_INPUT
    Code        : UDINT;
    ComponentID : UDINT;
END_VAR
VAR
    i : UDINT;
    j : UDINT;
END_VAR

FOR i := 1 TO _Count DO
    IF _Alarms[i].AlarmCode = Code AND 
       _Alarms[i].ComponentID = ComponentID THEN
        // Desplazar array hacia abajo
        FOR j := i TO _Count - 1 DO
            _Alarms[j] := _Alarms[j + 1];
        END_FOR
        // Limpiar último hueco
        _Alarms[_Count].ID          := 0;
        _Alarms[_Count].ComponentID := 0;
        _Alarms[_Count].AlarmCode   := 0;
        _Alarms[_Count].Active      := FALSE;
		_Alarms[_Count].Severity    := E_AlarmSeverity.SEVERITY_NONE;
		_Alarms[_Count].IsFault     := FALSE;
        _Alarms[_Count].Acknowledge := FALSE;
        _Count := _Count - 1;
        Clear := TRUE;
        RETURN;
    END_IF
END_FOR
Clear := FALSE;
```
### ClearAll

Resets the entire buffer — sets `Active = FALSE` on all entries and resets head, tail, and count. Used during component reset (`FB_ComponentBase.Reset`).
```pascal
METHOD ClearAll : BOOL
VAR
    i : UDINT;
END_VAR

FOR i := 1 TO GVL_Config.MAX_ALARMS_PER_COMPONENT DO
    _Alarms[i].ID          := 0;
    _Alarms[i].ComponentID := 0;
    _Alarms[i].AlarmCode   := 0;
    _Alarms[i].Active      := FALSE;
    _Alarms[i].Acknowledge := FALSE;
END_FOR
_Count := 0;
ClearAll := TRUE;
```

### Properties

| Property | Type | Description |
|---|---|---|
| `Count` | `UDINT` | Number of alarm records currently in the buffer (active and inactive). |
| `HasActive` | `BOOL` | TRUE if any alarm has `Active = TRUE`. |
| `HasFault` | `BOOL` | TRUE if any alarm has `Active = TRUE` AND `IsFault = TRUE`. |
| `Alarms` | `ARRAY OF ST_Alarm` | Read-only access to the full buffer. |

`HasFault` is the property that `FB_ComponentBase` exposes upward through `I_Component.HasFault`. A single boolean that tells the application layer whether any active alarm in this component requires an emergency response — no iteration needed at the application level.

---

## FB_ComponentBase — Alarm Generation

Components never write to `FB_AlarmQueue` directly. They call `SetAlarm`, which is the single controlled entry point for alarm generation at the component level.

```pascal
// SetAlarm — the only path to alarm generation in a component
METHOD SetAlarm : BOOL
VAR_INPUT
    Code     : UDINT;
	Severity : E_AlarmSeverity := E_AlarmSeverity.SEVERITY_WARNING;
END_VAR

SetAlarm := AlarmQueue.Push(
    Code        := Code,
	ComponentID := Info.ID,
    Severity    := Severity
);
```

This pattern enforces three things simultaneously: consistent `ST_Alarm` formatting across all components, automatic `ComponentID` injection, and immediate state machine response to fault-level alarms. None of these can be bypassed by a subclass.

`ClearAlarm` is the symmetric method — called when the fault condition resolves:

```pascal
METHOD ClearAlarm : BOOL
VAR_INPUT
    Code : UDINT;
END_VAR

ClearAlarm := AlarmQueue.Clear(
    Code        := Code,
	ComponentID := Info.ID
);
```

```pascal
METHOD AcknowledgeAlarm : BOOL
VAR_INPUT
    Code : UDINT;
END_VAR

AcknowledgeAlarm := AlarmQueue.Acknowledge(
    Code        := Code,
    ComponentID := Info.ID
);
```

---

## FB_AlarmManager — System-Wide Routing

`FB_AlarmManager` is the system-level alarm aggregator. It polls all registered components via `I_Diagnostic.GetAlarms`, consolidates new alarms into the system alarm list, and provides operator-level acknowledgement and filtering.

```
FUNCTION_BLOCK FB_AlarmManager IMPLEMENTS I_AlarmManager
VAR
    _Alarms         : ARRAY[1..GVL_Config.MAX_ALARMS] OF ST_Alarm;
    _AlarmCount     : UDINT := 0;
	_WriteIndex  : UDINT := 1;
END_VAR
```

### Update — The Polling Cycle

Called every PLC cycle from `MAIN`. Iterates all registered components, reads their `ActiveAlarms` property via `I_Diagnostic`, and copies new alarms (identified by `ID` comparison) into the system alarm list:

```pascal
METHOD Update : BOOL
VAR
    i            : UDINT;
    j            : UDINT;
    k            : UDINT;
    _comp            : POINTER TO FB_ComponentBase;  
    _compAlarms      : ARRAY[1..50] OF ST_Alarm;
    _compAlarmCount  : UDINT;
    _exists      : BOOL;
    _stillActive : BOOL;
END_VAR

FOR i := 1 TO GVL_Components.ComponentManager.ComponentCount DO
    _comp := GVL_Components.ComponentManager.GetComponentByID(i);
    
    IF _comp <> 0 THEN
        _compAlarms     := _comp^.ActiveAlarms;
        _compAlarmCount := _comp^.ActiveAlarmCount;

        // Dar de alta alarmas nuevas
        FOR k := 1 TO _compAlarmCount DO
            _exists := FALSE;
            // Buscar duplicados activos en todo el array
            FOR j := 1 TO GVL_Config.MAX_ALARMS DO
                IF _Alarms[j].AlarmCode   = _compAlarms[k].AlarmCode AND
                   _Alarms[j].ComponentID = _compAlarms[k].ComponentID AND
                   _Alarms[j].Active THEN
                    _exists := TRUE;
                END_IF
            END_FOR
            IF NOT _exists THEN
                // Circular — sobreescribe en WriteIndex
                _Alarms[_WriteIndex] := _compAlarms[k];
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

        // Dar de baja alarmas resueltas
        FOR j := 1 TO GVL_Config.MAX_ALARMS DO
            IF _Alarms[j].ComponentID = _comp^.ID AND _Alarms[j].Active THEN
                _stillActive := FALSE;
                FOR k := 1 TO _compAlarmCount DO
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

### AcknowledgeAll

Single operator action that clears the active alarm state across the entire system:

```pascal
METHOD AcknowledgeAll : BOOL
VAR
    i : UDINT;
END_VAR

FOR i := 1 TO _AlarmCount DO
    IF _Alarms[i].Active THEN
        _Alarms[i].Acknowledge := TRUE;
    END_IF
END_FOR
AcknowledgeAll := TRUE;
```

### AlarmsByComponent

Returns the alarm subset for a specific `ComponentID`. Used by HMI screens that display per-device alarm history — a pump detail screen shows only pump alarms, not system-wide alarms:

```pascal
METHOD AlarmsByComponent : UDINT
VAR_INPUT
    ComponentID : UDINT;
END_VAR
VAR
    i     : UDINT;
    count : UDINT := 0;
END_VAR

FOR i := 1 TO _AlarmCount DO
    IF _Alarms[i].ComponentID = ComponentID AND _Alarms[i].Active THEN
        count := count + 1;
    END_IF
END_FOR
AlarmsByComponent := count;

```

---

## The Complete Propagation Chain

From fault detection to operator acknowledgement, every step is automatic:

```
1. Fault condition detected in component
   (watchdog timeout, threshold violation, hardware error)
        ↓
2. Component calls SetAlarm(AlarmCode, Severity, IsFault)
        ↓
3. FB_ComponentBase builds ST_Alarm and calls _AlarmQueue.Push
        ↓
4. FB_AlarmQueue assigns ID, records Timestamp, stores in buffer
        ↓
5. If IsFault = TRUE → _State transitions to STATE_FAULT immediately
        ↓
6. I_Component.HasFault = TRUE (read by application layer if needed)
        ↓
7. FB_AlarmManager.Update polls I_ComponentManager.ActiveAlarms next cycle
```

Steps 2 through 8 require zero application-layer code. The only lines written outside the framework are the fault detection condition (step 1) and the Reset command (step 12).

---

## Alarm Lifecycle

An alarm passes through four states from generation to resolution:

```
INACTIVE ──[SetAlarm]──► ACTIVE
                            │
                     [AcknowledgeAll]
                            │
                            ▼
                       ACKNOWLEDGED (still Active = TRUE)
                            │
                      [ClearAlarm]
                            │
                            ▼
                  INACTIVE + ACKNOWLEDGED
                  (record retained in buffer)
```

**Active but unacknowledged** — fault is present, operator has not responded. `HasActive = TRUE`, `HasFault = TRUE` (if IsFault). HMI shows active alarm indicator.

**Active and acknowledged** — operator has seen the alarm but the fault condition has not cleared. `Acknowledge = TRUE` but `Active = TRUE`. Component remains in fault state.

**Inactive and acknowledged** — fault condition resolved, operator acknowledged. Record retained in buffer for history. Component can accept `CMD_RESET`.

**Inactive and unacknowledged** — possible if `ClearAlarm` is called before the operator acknowledges (e.g., a transient fault that self-resolved). Record retained. No reset required.

---

## OPC UA Integration

`ST_Alarm` and `ST_ComponentStatus` are designed for direct OPC UA node exposure. Both TwinCAT and CODESYS OPC UA servers can expose any `STRUCT` instance as a structured node without manual mapping.

The `E_EventType` enum maps directly to OPC UA `EventType` categories. `EVENT_ALARM_RAISED` and `EVENT_ALARM_CLEARED` correspond to OPC UA `AlarmConditionType` transitions. `EVENT_STATE_CHANGE` corresponds to `ProgramStateMachineType` transitions.

For OPC UA `EventNotifier` emission, `FB_AlarmManager` is the natural integration point — it holds the consolidated system alarm list and can emit `EVENT_ALARM_RAISED` events when new alarms are merged, and `EVENT_ALARM_CLEARED` events when `Active` transitions to FALSE.

This integration is part of the v0.3 roadmap.

---

## Design Decisions

**`IsFault` separate from `Severity`.**
An earlier design used severity level alone to determine fault state — `SEVERITY_CRITICAL` and above triggered `STATE_FAULT`. This was changed because some critical alarms (e.g., a communication warning on a non-essential sensor) should not halt the machine. `IsFault` gives the component author explicit control over which alarms trigger emergency stop, independent of the severity level used for display and logging.

**Oldest-alarm overwrite when buffer is full.**
The alternative is to reject new alarms when the buffer is full (newest-rejected). Oldest-overwrite was chosen because in a fault cascade — many alarms generated rapidly — the most recent alarms are typically more diagnostically relevant than the original trigger. The root cause is usually still present and will re-alarm; the cascade alarms that followed it are more useful for understanding the fault sequence.

**`AlarmCode` as `UDINT`, not an enum.**
A system-wide alarm code enum would require every component class to share a single enum, creating tight coupling between unrelated components. Using `UDINT` with per-class enums (`E_AnalogAlarm`, actuator-specific enums) keeps alarm codes in their natural namespace and allows each component type to evolve its alarm vocabulary independently.

**Monotonic `_NextID` rather than reset-on-clear.**
When `ClearAll` is called during `Reset`, `_NextID` is not reset. This means `FB_AlarmManager` can always identify truly new alarms by ID comparison — a re-triggered alarm after reset gets a new ID and is correctly treated as a new event, not a duplicate of the pre-reset alarm.

---
