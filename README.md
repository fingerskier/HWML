# HWML
HardWare Modeling Language

# HWML: Hardware Modeling Language

## Abstract

HWML is a declarative, JSON-based language for modeling hardware systems as reactive dataflow graphs.  It enables composition of hardware modules, automatic dependency resolution, and seamless switching between physical hardware and simulation.

The structure is analogous to a spreadsheet- where hardware modules are like sheets and the components are cells.

## Design Goals

1. **Declarative** - Describe relationships, not procedures
2. **Composable** - Modules combine without glue code
3. **Portable** - Pure JSON serialization, no code in data
4. **Simulable** - Same model runs against hardware or physics simulation
5. **Verifiable** - Static analysis of constraints, units, bounds
6. **Versionable** - Diff-friendly, merge-friendly structure

## Document Structure

A HWML document is a JSON object where each top-level key is a **Module**.

```json
{
  "name": { ... },
  "components": { ... }
}
```

### Reserved Names

| Name | Purpose |
|------|---------|
| `_meta` | Document metadata |
| `_config` | Runtime configuration |
| `_types` | Custom type definitions |
| `_imports` | Module dependencies |
| `_modules` | Nested submodule declarations |
| `_params` | Module parameter definitions |
| `_wiring` | Component interconnections |

## Modules

A Module is a JSON object where each key is a **Component** name.

```json
{
  "loadCell": {
    "rawAdc": { ... },
    "force": { ... }
  }
}
```

### Module Semantics

- Modules are **namespaces** for cells
- Modules can contain other modules (hierarchical composition)
- Modules may represent: hardware modules, controllers, safety systems, simulations, UI bindings
- All modules evaluate together in a single reactive graph

## Module Composition & File Structure

HWML supports hierarchical composition where modules can import and embed other modules. Module names mirror their file paths.

### Directory Structure

```
project/
├── _types.hwml.json          # shared types
├── _config.hwml.json         # runtime config
├── sensors/
│   ├── loadCell.hwml.json
│   └── encoder.hwml.json
├── actuators/
│   └── linear.hwml.json
├── control/
│   ├── pid.hwml.json
│   └── safety.hwml.json
└── main.hwml.json            # root module
```

### Module Naming Convention

Module names mirror their file path (without extension):

| File Path | Module Name | Reference Syntax |
|-----------|-------------|------------------|
| `sensors/loadCell.hwml.json` | `sensors/loadCell` | `sensors/loadCell.force` |
| `control/pid.hwml.json` | `control/pid` | `control/pid.output` |
| `main.hwml.json` | `main` | `main.enabled` |

### Import Syntax

Modules declare dependencies via `_imports`:

```json
{
  "_imports": {
    "value": [
      "sensors/loadCell",
      "sensors/encoder",
      "control/pid"
    ]
  }
}
```

Or with aliasing for shorthand references:

```json
{
  "_imports": {
    "value": {
      "lc": "sensors/loadCell",
      "enc": "sensors/encoder",
      "pid": "control/pid"
    }
  }
}
```

With aliases, you can use `lc.force` instead of `sensors/loadCell.force`.

### Nested Modules (Inline Composition)

A module can embed another module as a submodule using `_modules`:

```json
{
  "robotArm": {
    "_modules": {
      "shoulder": { "use": "joints/rotary", "config": { "maxTorque": 100 } },
      "elbow": { "use": "joints/rotary", "config": { "maxTorque": 50 } },
      "wrist": { "use": "joints/rotary", "config": { "maxTorque": 20 } }
    },

    "totalPower": {
      "formula": "shoulder.power + elbow.power + wrist.power"
    }
  }
}
```

Reference syntax for nested modules:
- `robotArm/shoulder.position` - from outside
- `shoulder.position` - from within `robotArm`

### Module Parameters

Reusable modules declare parameters via `_params`:

**joints/rotary.hwml.json:**
```json
{
  "_params": {
    "maxTorque": { "type": "float", "units": "Nm", "required": true },
    "gearRatio": { "type": "float", "default": 1.0 },
    "inverted": { "type": "bool", "default": false }
  },

  "encoder": { "input": "hardware", "type": "int" },
  "position": {
    "formula": "inverted ? -encoder / gearRatio : encoder / gearRatio",
    "units": "deg"
  },
  "torque": { "input": "hardware", "units": "Nm", "range": [0, "maxTorque"] },
  "overload": { "formula": "torque > maxTorque" }
}
```

### Updated Reference Syntax

| Pattern | Meaning |
|---------|---------|
| `name` | Component in same module |
| `submodule.name` | Component in child module |
| `../name` | Component in parent module |
| `path/to/module.name` | Absolute path reference |
| `alias.name` | Aliased import reference |

### Resolution Rules

1. **Local first** - check current module
2. **Child modules** - check `_modules` entries
3. **Imports** - check `_imports` (aliases first, then full paths)
4. **Parent scope** - if nested, check parent module
5. **Error** - unresolved reference fails at load time

### Root Module Discovery

The runtime loads from an entry point (e.g., `main.hwml.json`) and recursively resolves imports. All paths are relative to the project root.

```bash
hwml run ./main.hwml.json
hwml run ./project/           # looks for main.hwml.json or index.hwml.json
```

### Parameterized Module Example

**sensors/loadCell.hwml.json:**
```json
{
  "_params": {
    "calibration": { "type": "float", "units": "lbs/count", "required": true },
    "tare": { "type": "int", "default": 0 }
  },

  "rawAdc": { "input": "hardware", "type": "int", "range": [0, 4095] },
  "force": {
    "formula": "max(0, (rawAdc - tare) * calibration)",
    "units": "lbs"
  }
}
```

**control/pid.hwml.json:**
```json
{
  "_params": {
    "kp": { "type": "float", "default": 1.0 },
    "ki": { "type": "float", "default": 0.0 },
    "kd": { "type": "float", "default": 0.0 },
    "clampIntegral": { "type": "float", "default": 1000 }
  },

  "setpoint": { "input": "parent", "type": "float" },
  "feedback": { "input": "parent", "type": "float" },

  "error": { "formula": "setpoint - feedback" },
  "integral": {
    "formula": "clamp(prev.integral + error * dt, -clampIntegral, clampIntegral)",
    "state": true
  },
  "derivative": {
    "formula": "dt > 0 ? (error - prev.error) / dt : 0",
    "state": true
  },
  "output": {
    "formula": "kp * error + ki * integral + kd * derivative"
  }
}
```

**main.hwml.json:**
```json
{
  "_imports": {
    "value": ["control/safety"]
  },

  "_modules": {
    "load": {
      "use": "sensors/loadCell",
      "config": { "calibration": 0.245, "tare": 892 }
    },
    "pid": {
      "use": "control/pid",
      "config": { "kp": 0.8, "ki": 0.05, "kd": 0.02 },
      "bind": {
        "setpoint": "main.targetForce",
        "feedback": "main/load.force"
      }
    }
  },

  "targetForce": { "input": "user", "value": 0, "units": "lbs" },
  "output": {
    "formula": "control/safety.enabled ? pid.output : 0",
    "output": "hardware"
  }
}
```

## Components

A Component is the fundamental unit of computation in HWML. Each component has three parts:

1. **Inputs** - Named input ports that receive values from external sources or other components
2. **Outputs** - Named output ports that emit computed values
3. **Nodes** - An internal state-machine DAG (directed acyclic graph) that maps inputs to outputs

### Component Structure

```json
{
  "componentName": {
    "inputs": {
      "portName": { "type": "float", "units": "lbs", "source": "hardware" },
      "otherPort": { "type": "int", "default": 0 }
    },
    "outputs": {
      "portName": { "type": "float", "units": "N", "target": "hardware" },
      "status": { "type": "bool", "target": "indicator" }
    },
    "nodes": {
      "nodeName": { "formula": "...", "state": true },
      "anotherNode": { "formula": "..." }
    }
  }
}
```

### Inputs

Input ports declare how data enters the component.

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Data type: `int`, `float`, `bool`, `string` |
| `units` | string | Physical units (optional) |
| `source` | string | Input source: `hardware`, `user`, `network`, `parent`, or omit for component references |
| `default` | any | Default value if not connected |
| `range` | [min, max] | Valid input range (optional) |
| `desc` | string | Human-readable description |

```json
{
  "inputs": {
    "rawAdc": { "type": "int", "source": "hardware", "range": [0, 4095] },
    "setpoint": { "type": "float", "source": "user", "units": "lbs", "default": 0 },
    "feedback": { "type": "float", "units": "lbs" },
    "enabled": { "type": "bool", "default": true }
  }
}
```

### Outputs

Output ports declare what data the component produces and where it goes.

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Data type: `int`, `float`, `bool`, `string` |
| `units` | string | Physical units (optional) |
| `target` | string | Output destination: `hardware`, `indicator`, `log`, `network`, or omit for internal use |
| `from` | string | Node name that provides this output's value |
| `clamp` | [min, max] | Output clamping (optional) |
| `desc` | string | Human-readable description |

```json
{
  "outputs": {
    "force": { "type": "float", "units": "lbs", "from": "calibratedForce", "clamp": [0, 2000] },
    "command": { "type": "float", "target": "hardware", "from": "clampedOutput" },
    "fault": { "type": "bool", "target": "indicator", "from": "faultLatched" }
  }
}
```

### Nodes (Internal State-Machine DAG)

Nodes form the internal computation graph that transforms inputs into outputs. Each node computes a value from inputs, other nodes, or previous state.

| Property | Type | Description |
|----------|------|-------------|
| `formula` | string | Expression computing this node's value |
| `value` | any | Static constant value (mutually exclusive with formula) |
| `state` | bool | If true, node maintains state across ticks (enables `prev.` access) |
| `type` | string | Explicit type (inferred if omitted) |
| `units` | string | Physical units (optional) |

```json
{
  "nodes": {
    "calibration": { "value": 0.245, "units": "lbs/count" },
    "tare": { "value": 892 },
    "calibratedForce": { "formula": "max(0, (rawAdc - tare) * calibration)" },

    "error": { "formula": "setpoint - feedback" },
    "integral": {
      "formula": "clamp(prev.integral + error * dt, -500, 500)",
      "state": true
    },
    "derivative": {
      "formula": "dt > 0 ? (error - prev.error) / dt : 0",
      "state": true
    },
    "rawOutput": { "formula": "kp * error + ki * integral + kd * derivative" },
    "clampedOutput": { "formula": "enabled ? clamp(rawOutput, -100, 100) : 0" }
  }
}
```

### Node Reference Rules

Within a component's node formulas:

| Reference | Meaning |
|-----------|---------|
| `inputName` | Value from an input port |
| `nodeName` | Value from another node (must be acyclic) |
| `prev.nodeName` | Previous tick's value (requires `state: true` on that node) |

### Complete Component Example

```json
{
  "pidController": {
    "inputs": {
      "setpoint": { "type": "float", "source": "user", "units": "lbs", "default": 0 },
      "feedback": { "type": "float", "units": "lbs" },
      "enabled": { "type": "bool", "default": true }
    },
    "outputs": {
      "command": { "type": "float", "target": "hardware", "from": "output", "clamp": [-100, 100] },
      "error": { "type": "float", "from": "error" },
      "fault": { "type": "bool", "target": "indicator", "from": "saturated" }
    },
    "nodes": {
      "kp": { "value": 0.8 },
      "ki": { "value": 0.05 },
      "kd": { "value": 0.02 },
      "integralLimit": { "value": 500 },

      "error": { "formula": "setpoint - feedback" },
      "integral": {
        "formula": "clamp(prev.integral + error * dt, -integralLimit, integralLimit)",
        "state": true
      },
      "derivative": {
        "formula": "dt > 0 ? (error - prev.error) / dt : 0",
        "state": true
      },
      "rawOutput": { "formula": "kp * error + ki * integral + kd * derivative" },
      "output": { "formula": "enabled ? rawOutput : 0" },
      "saturated": { "formula": "abs(rawOutput) > 100" }
    }
  }
}
```

### Simplified Component Syntax

For simple components with a single output, a shorthand syntax is available:

```json
{
  "loadCell": {
    "inputs": {
      "rawAdc": { "type": "int", "source": "hardware", "range": [0, 4095] }
    },
    "output": { "type": "float", "units": "lbs" },
    "nodes": {
      "calibration": { "value": 0.245 },
      "tare": { "value": 892 },
      "force": { "formula": "max(0, (rawAdc - tare) * calibration)" }
    },
    "result": "force"
  }
}
```

When `output` (singular) and `result` are used, the component's value can be referenced directly as `loadCell` instead of `loadCell.force`.

### Component DAG Semantics

The internal nodes form a DAG with these properties:

1. **Acyclic** - No node can depend on itself (except via `prev.` for state)
2. **Deterministic** - Given the same inputs and previous state, outputs are identical
3. **Ordered** - Nodes evaluate in topological order respecting dependencies
4. **Isolated** - Nodes cannot reference other components directly; all external data flows through inputs

## Component Wiring

Components are connected via the `_wiring` section, which declares how outputs flow to inputs.

### Wiring Syntax

```json
{
  "_wiring": {
    "connections": [
      { "from": "componentA.outputName", "to": "componentB.inputName" },
      { "from": "sensor.value", "to": "controller.feedback" },
      { "from": "controller.command", "to": "actuator.commandIn" }
    ]
  }
}
```

### Connection Properties

| Property | Type | Description |
|----------|------|-------------|
| `from` | string | Source: `componentName.outputName` |
| `to` | string | Destination: `componentName.inputName` |
| `transform` | string | Optional formula to transform the value |

### Fan-out and Fan-in

- **Fan-out**: One output can connect to multiple inputs
- **Fan-in**: Multiple outputs can connect to the same input (last write wins, or use a combiner node)

```json
{
  "_wiring": {
    "connections": [
      { "from": "loadCell.force", "to": "controller.feedback" },
      { "from": "loadCell.force", "to": "safety.forceInput" },
      { "from": "loadCell.force", "to": "display.forceValue" }
    ]
  }
}
```

### Implicit Wiring

For simple cases, inputs can reference outputs directly using dot notation in the `bind` property:

```json
{
  "controller": {
    "inputs": {
      "feedback": { "type": "float", "bind": "loadCell.force" }
    }
  }
}
```

This is equivalent to an explicit `_wiring` connection.

## Functions

Functions are expressions within node formulas. They can reference inputs, other nodes, and previous state.

### Node Formula References

| Pattern | Meaning |
|---------|---------|
| `name` | Input port or another node in the same component |
| `prev.name` | Previous tick's value of a stateful node |

### Language

Functions are just strings.  They can be any lamguage you need- e.g. python snippets could be transpiled into a full script that implements the hardware module.

### Recommended Variables

| Variable | Type | Description |
|----------|------|-------------|
| `dt` | float | Time delta since last tick (seconds) |
| `time` | float | Total elapsed time (seconds) |
| `tick` | int | Current tick number |
| `PI` | float | 3.14159... |
| `E` | float | 2.71828... |

### Formula Examples

```json
{
  "error": { "formula": "setpoint - feedback" },
  
  "integral": { 
    "formula": "clamp(prev.integral + error * dt, -1000, 1000)", 
    "state": true 
  },
  
  "output": { 
    "formula": "enabled ? kp * error + ki * integral : 0" 
  },
  
  "velocity": { 
    "formula": "(position - prev.position) / dt" 
  },
  
  "faultLatched": { 
    "formula": "fault || prev.faultLatched", 
    "state": true 
  },
  
  "smoothedForce": {
    "formula": "lowpass(rawForce, prev.smoothedForce, 0.1)",
    "state": true
  }
}
```

## Evaluation Model

### Tick Cycle

Each evaluation tick proceeds as follows:

1. **Inject External Inputs** - Hardware, user, and network inputs are written to component input ports
2. **Resolve Component Order** - Topologically sort components based on wiring dependencies
3. **Evaluate Each Component**:
   - Copy wired inputs from upstream component outputs
   - Build internal node dependency graph
   - Topologically sort nodes within the component
   - Evaluate each node formula in order
   - Apply output constraints (clamp, range validation)
4. **Snapshot State** - Copy stateful nodes to `prev` buffer
5. **Emit Outputs** - Send outputs marked with `target` to hardware/indicators/logs
6. **Advance Time** - Increment `time` and `tick`, store `dt`

### Two-Level DAG

The system has two dependency levels:

1. **System-level DAG** - Components ordered by wiring connections
2. **Component-level DAG** - Nodes ordered by formula references within each component

Both levels must be acyclic (except for `prev.` references within component nodes).

### Dependency Rules

- Node self-reference requires `prev.` prefix
- Circular dependencies at either level are forbidden
- The runtime MUST detect cycles and report them with the full path

### Evaluation Guarantees

- Each component evaluates exactly once per tick
- Each node within a component evaluates exactly once
- Components see consistent outputs from upstream components
- `prev` values are from the completed previous tick
- First tick: `prev` values are initialized to `value` if present, else `0`/`false`/`""`

### Error Handling

| Error | Behavior |
|-------|----------|
| Undefined input/node reference | Load-time error |
| Unconnected input (no default) | Load-time error |
| Circular dependency (system or component) | Load-time error |
| Division by zero | Runtime: `NaN` or `Infinity` (IEEE 754) |
| Out of range | Runtime: warning emitted, value clamped if `clamp` specified |
| Type mismatch | Runtime: coercion attempted, else `NaN` |

## Metadata Sheet

The optional `_meta` sheet documents the model:

```json
{
  "_meta": {
    "name": { "value": "OsteoStrong Actuator Controller" },
    "version": { "value": "2.1.0" },
    "author": { "value": "Matt" },
    "created": { "value": "2025-01-15" },
    "modified": { "value": "2025-01-17" },
    "description": { "value": "Force-feedback control for linear actuator" },
    "tickRate": { "value": 100, "units": "Hz" },
    "license": { "value": "Proprietary" }
  }
}
```

## Configuration Module

The optional `_config` sheet controls runtime behavior:

```json
{
  "_config": {
    "tickRate": { "value": 100, "units": "Hz", "desc": "Target evaluation rate" },
    "maxTickTime": { "value": 5, "units": "ms", "desc": "Warn if tick exceeds" },
    "allowNaN": { "value": false, "desc": "If false, NaN triggers fault" },
    "logLevel": { "value": "warn", "enum": ["debug", "info", "warn", "error"] },
    "simMode": { "value": false, "desc": "Use simulation sheets for inputs" }
  }
}
```

## Type Definitions Sheet

The optional `_types` sheet defines custom compound types:

```json
{
  "_types": {
    "PIDGains": {
      "value": {
        "fields": {
          "kp": { "type": "float", "default": 1.0 },
          "ki": { "type": "float", "default": 0.0 },
          "kd": { "type": "float", "default": 0.0 }
        }
      }
    },
    "Vector3": {
      "value": {
        "array": "float",
        "length": 3
      }
    }
  }
}
```

## Simulation Binding

Simulation sheets provide physics models that substitute for hardware inputs.

### Convention

- Simulation sheet names are prefixed with `sim_`
- They contain `state: true` cells for physical quantities
- They read from controller outputs and write to modeled sensor values

### Binding Modes

In the `_config` sheet:

```json
{
  "_config": {
    "simMode": { "value": true },
    "simBindings": {
      "value": {
        "actuator.position": "sim_actuator.position",
        "actuator.velocity": "sim_actuator.velocity",
        "loadCell.rawAdc": "sim_loadCell.rawAdc"
      }
    }
  }
}
```

When `simMode` is true, input components read from their bound simulation components instead of hardware.

### Example Simulation Module

```json
{
  "sim_actuator": {
    "inputs": {
      "command": { "type": "float", "desc": "Motor command from controller" },
      "externalLoad": { "type": "float", "units": "N", "default": 0 }
    },
    "outputs": {
      "position": { "type": "float", "units": "m", "from": "position" },
      "velocity": { "type": "float", "units": "m/s", "from": "velocity" }
    },
    "nodes": {
      "mass": { "value": 2.5, "units": "kg" },
      "friction": { "value": 15, "units": "N/(m/s)" },
      "motorConstant": { "value": 50, "units": "N/%" },
      "springK": { "value": 0, "units": "N/m", "desc": "External spring load" },

      "motorForce": { "formula": "command * motorConstant", "units": "N" },
      "frictionForce": { "formula": "-friction * velocity", "units": "N" },
      "springForce": { "formula": "-springK * position", "units": "N" },
      "netForce": { "formula": "motorForce + frictionForce + springForce + externalLoad", "units": "N" },
      "acceleration": { "formula": "netForce / mass", "units": "m/s²" },
      "velocity": {
        "formula": "clamp(prev.velocity + acceleration * dt, -0.5, 0.5)",
        "units": "m/s",
        "state": true
      },
      "position": {
        "formula": "clamp(prev.position + velocity * dt, 0, 0.5)",
        "units": "m",
        "state": true
      }
    }
  }
}
```

## Hardware Binding

Hardware adapters map cells to physical I/O.

### Adapter Interface (Informative)

```typescript
interface HardwareAdapter {
  // Called before tick to gather inputs
  readInputs(): Map<string, CellValue>;
  
  // Called after tick to send outputs
  writeOutputs(outputs: Map<string, CellValue>): void;
  
  // Lifecycle
  connect(): Promise<void>;
  disconnect(): Promise<void>;
}
```

### Hardware Mapping Convention

Input ports with `source: "hardware"` declare their hardware source via the `hw` property:

```json
{
  "inputs": {
    "encoderCount": {
      "type": "int",
      "source": "hardware",
      "hw": { "channel": "encoder0", "countsPerRev": 4096 }
    },
    "limitSwitch": {
      "type": "bool",
      "source": "hardware",
      "hw": { "channel": "gpio12", "activeHigh": false }
    }
  }
}
```

Output ports with `target: "hardware"` declare their hardware destination:

```json
{
  "outputs": {
    "pwmCommand": {
      "type": "float",
      "target": "hardware",
      "from": "clampedOutput",
      "hw": { "channel": "pwm0", "range": [0, 4095] }
    },
    "enablePin": {
      "type": "bool",
      "target": "hardware",
      "from": "enabled",
      "hw": { "channel": "gpio5" }
    }
  }
}
```

The `hw` object is adapter-specific and opaque to the HWML runtime.

## Units (Informative)

Units are strings for documentation and optional dimensional analysis. Recommended format:

| Unit | Meaning |
|------|---------|
| `mm`, `m`, `in`, `ft` | Length |
| `mm/s`, `m/s` | Velocity |
| `mm/s²`, `m/s²` | Acceleration |
| `N`, `lbs`, `kg` | Force (kg treated as kgf) |
| `Nm`, `lb-ft` | Torque |
| `deg`, `rad` | Angle |
| `deg/s`, `rad/s`, `rpm` | Angular velocity |
| `ms`, `s` | Time |
| `Hz` | Frequency |
| `%` | Percentage (0-100) |
| `count` | Dimensionless integer |
| `V`, `mV` | Voltage |
| `A`, `mA` | Current |

Compound units use `/` and `*`: `lbs/count`, `N*m`, `m/s²`

A conforming implementation MAY perform dimensional analysis and warn on unit mismatches.

## Validation Requirements

A conforming implementation MUST validate:

1. **Schema** - Document conforms to JSON and component schema (inputs, outputs, nodes)
2. **Input references** - All component input connections resolve to valid output ports
3. **Output mappings** - Each output's `from` field references a valid node within the component
4. **Node references** - All references within node formulas resolve to inputs or other nodes
5. **Internal DAG** - No circular dependencies within a component's nodes (except via `prev.`)
6. **System DAG** - No circular dependencies in the component wiring graph
7. **Node exclusivity** - Each node has exactly one of: `value` or `formula`
8. **Types** - Formula return types are compatible with declared types

A conforming implementation SHOULD validate:

1. **Units** - Dimensional consistency in formulas
2. **Ranges** - Input/output values within declared ranges
3. **Connectivity** - Warn on unconnected inputs without defaults
4. **Reachability** - Warn on nodes that don't contribute to any output

## File Extension

HWML documents use the `.hwml.json` extension, or simply `.hwml` if tooling supports it.

## MIME Type

`application/vnd.hwml+json`

## Example: Complete System

```json
{
  "_meta": {
    "name": { "value": "Force Feedback Actuator" },
    "version": { "value": "1.0.0" },
    "tickRate": { "value": 100, "units": "Hz" }
  },

  "loadCell": {
    "inputs": {
      "rawAdc": { "type": "int", "source": "hardware", "range": [0, 4095] }
    },
    "outputs": {
      "force": { "type": "float", "units": "lbs", "from": "force", "clamp": [0, 2000] }
    },
    "nodes": {
      "calibration": { "value": 0.245, "units": "lbs/count" },
      "tare": { "value": 892, "units": "count" },
      "force": { "formula": "max(0, (rawAdc - tare) * calibration)" }
    }
  },

  "controller": {
    "inputs": {
      "setpoint": { "type": "float", "source": "user", "units": "lbs", "default": 0, "range": [0, 800] },
      "feedback": { "type": "float", "units": "lbs" },
      "safetyEnabled": { "type": "bool", "default": true }
    },
    "outputs": {
      "command": { "type": "float", "target": "hardware", "from": "output", "clamp": [-100, 100] },
      "error": { "type": "float", "from": "error" }
    },
    "nodes": {
      "kp": { "value": 0.8 },
      "ki": { "value": 0.05 },
      "kd": { "value": 0.02 },

      "error": { "formula": "setpoint - feedback" },
      "integral": {
        "formula": "clamp(prev.integral + error * dt, -500, 500)",
        "state": true
      },
      "derivative": {
        "formula": "dt > 0 ? (error - prev.error) / dt : 0",
        "state": true
      },
      "rawOutput": { "formula": "kp * error + ki * integral + kd * derivative" },
      "output": { "formula": "safetyEnabled ? rawOutput : 0" }
    }
  },

  "actuator": {
    "inputs": {
      "position": { "type": "float", "source": "hardware", "units": "mm" },
      "commandIn": { "type": "float" }
    },
    "outputs": {
      "velocity": { "type": "float", "units": "mm/s", "from": "velocity" },
      "command": { "type": "float", "target": "hardware", "from": "commandIn" }
    },
    "nodes": {
      "velocity": {
        "formula": "dt > 0 ? (position - prev.position) / dt : 0",
        "state": true
      }
    }
  },

  "safety": {
    "inputs": {
      "force": { "type": "float", "units": "lbs" },
      "position": { "type": "float", "units": "mm" },
      "velocity": { "type": "float", "units": "mm/s" },
      "reset": { "type": "bool", "source": "user", "default": false }
    },
    "outputs": {
      "enabled": { "type": "bool", "target": "indicator", "from": "enabled" },
      "faultLatched": { "type": "bool", "target": "indicator", "from": "faultLatched" }
    },
    "nodes": {
      "maxForce": { "value": 800, "units": "lbs" },
      "maxPosition": { "value": 450, "units": "mm" },
      "maxVelocity": { "value": 100, "units": "mm/s" },

      "overload": { "formula": "force > maxForce" },
      "overtravel": { "formula": "position > maxPosition" },
      "overspeed": { "formula": "abs(velocity) > maxVelocity" },

      "fault": { "formula": "overload || overtravel || overspeed" },
      "faultLatched": {
        "formula": "reset ? false : (fault || prev.faultLatched)",
        "state": true
      },
      "enabled": { "formula": "!faultLatched" }
    }
  },

  "sim_physics": {
    "inputs": {
      "command": { "type": "float" },
      "active": { "type": "bool", "default": false }
    },
    "outputs": {
      "position": { "type": "float", "units": "mm", "from": "position" },
      "velocity": { "type": "float", "units": "m/s", "from": "velocity" }
    },
    "nodes": {
      "mass": { "value": 5, "units": "kg" },
      "motorK": { "value": 2, "units": "N/%" },
      "damping": { "value": 50, "units": "N/(m/s)" },

      "motorForce": { "formula": "command * motorK", "units": "N" },
      "dampingForce": { "formula": "-damping * velocity", "units": "N" },
      "acceleration": { "formula": "(motorForce + dampingForce) / mass", "units": "m/s²" },
      "velocity": {
        "formula": "active ? prev.velocity + acceleration * dt : 0",
        "state": true,
        "units": "m/s"
      },
      "position": {
        "formula": "active ? clamp(prev.position + velocity * dt * 1000, 0, 500) : 0",
        "state": true,
        "units": "mm"
      }
    }
  },

  "_config": {
    "tickRate": { "value": 100, "units": "Hz" },
    "simMode": { "value": false },
    "simBindings": {
      "value": {
        "actuator.position": "sim_physics.position",
        "loadCell.rawAdc": "sim_physics.simAdc"
      }
    }
  },

  "_wiring": {
    "desc": "Component interconnections",
    "connections": [
      { "from": "loadCell.force", "to": "controller.feedback" },
      { "from": "loadCell.force", "to": "safety.force" },
      { "from": "actuator.position", "to": "safety.position" },
      { "from": "actuator.velocity", "to": "safety.velocity" },
      { "from": "safety.enabled", "to": "controller.safetyEnabled" },
      { "from": "controller.command", "to": "actuator.commandIn" },
      { "from": "controller.command", "to": "sim_physics.command" },
      { "from": "_config.simMode", "to": "sim_physics.active" }
    ]
  }
}
```

## Appendix A: Grammar (Informative)

Formula grammar in EBNF:

```ebnf
formula     = ternary ;
ternary     = or ( "?" ternary ":" ternary )? ;
or          = and ( "||" and )* ;
and         = equality ( "&&" equality )* ;
equality    = comparison ( ( "==" | "!=" ) comparison )* ;
comparison  = term ( ( "<" | ">" | "<=" | ">=" ) term )* ;
term        = factor ( ( "+" | "-" ) factor )* ;
factor      = exponent ( ( "*" | "/" | "%" ) exponent )* ;
exponent    = unary ( "**" unary )* ;
unary       = ( "!" | "-" ) unary | call ;
call        = primary ( "(" arguments? ")" )* ;
arguments   = formula ( "," formula )* ;
primary     = NUMBER | STRING | BOOL | reference | "(" formula ")" ;
reference   = ( "prev" "." )? IDENT ( "." IDENT )* ;
```
