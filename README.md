# HWML
HardWare Modeling Language

# HWML: Hardware Modeling Language

## Abstract

HWML is a declarative, JSON-based language for modeling hardware systems as reactive dataflow graphs.  It enables composition of hardware modules, automatic dependency resolution, and seamless switching between physical hardware and simulation.

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

### Reserved Sheet Names

| Name | Purpose |
|------|---------|
| `_meta` | Document metadata |
| `_config` | Runtime configuration |
| `_types` | Custom type definitions |

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
- Modules have no inheritance or nesting (flat structure)
- Modules may represent: hardware modules, controllers, safety systems, simulations, UI bindings
- All modules evaluate together in a single reactive graph

## Components

A Component is the fundamental unit. It holds either a static value, a computed formula, or an external input.

### Component Examples

```json
{
  "calibration": { 
    "value": 0.245, 
    "units": "lbs/count",
    "desc": "Load cell calibration factor"
  },
  
  "force": { 
    "formula": "(rawAdc - tare) * calibration", 
    "units": "lbs",
    "clamp": [0, 2000]
  },
  
  "rawAdc": { 
    "input": "hardware", 
    "type": "int", 
    "range": [0, 4095] 
  },
  
  "fault": {
    "function": "force > maxForce",
    "type": "bool",
    "output": "indicator"
  }
}
```

## Functions

Functions are expressions describing component behavior.  Other components are referenced by name.

### References Syntax

| Pattern | Meaning |
|---------|---------|
| `name` | Component in same sheet |
| `module.name` | Component in another module |

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

1. **Inject Inputs** - Hardware, user, and network inputs are written to input components
2. **Build Dependency Graph** - Parse formulas, extract references
3. **Topological Sort** - Order components so dependencies evaluate first
4. **Evaluate** - Compute each formula cell in order
5. **Apply Constraints** - Clamp values, validate ranges
6. **Snapshot State** - Copy state components to `prev` buffer
7. **Emit Outputs** - Send output components to hardware/indicators/logs
8. **Advance Time** - Increment `t` and `tick`, store `dt`

### Dependency Rules

- Self-reference for state requires `prev.` prefix
- Circular dependencies are forbidden and must error at load time
- The runtime MUST detect cycles and report them with the full cycle path

### Evaluation Guarantees

- Components evaluate exactly once per tick
- Evaluation order respects dependencies
- All components see a consistent snapshot (no partial updates visible)
- `prev` values are from the completed previous tick
- First tick: `prev` values are initialized to `value` if present, else `0`/`false`/`""`

### Error Handling

| Error | Behavior |
|-------|----------|
| Undefined reference | Load-time error |
| Circular dependency | Load-time error |
| Division by zero | Runtime: `NaN` or `Infinity` (IEEE 754) |
| Out of range (with range) | Runtime: warning emitted, value unchanged |
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
    "_desc": { "value": "Linear actuator physics model" },
    
    "mass": { "value": 2.5, "units": "kg" },
    "friction": { "value": 15, "units": "N/(m/s)" },
    "motorConstant": { "value": 50, "units": "N/%" },
    "springK": { "value": 0, "units": "N/m", "desc": "External spring load" },
    
    "motorForce": { 
      "formula": "actuator.command * motorConstant", 
      "units": "N" 
    },
    "frictionForce": { 
      "formula": "-friction * velocity", 
      "units": "N" 
    },
    "springForce": {
      "formula": "-springK * position",
      "units": "N"
    },
    "netForce": { 
      "formula": "motorForce + frictionForce + springForce", 
      "units": "N" 
    },
    "acceleration": { 
      "formula": "netForce / mass", 
      "units": "m/s²" 
    },
    "velocity": { 
      "formula": "prev.velocity + acceleration * dt", 
      "units": "m/s",
      "state": true,
      "clamp": [-0.5, 0.5]
    },
    "position": { 
      "formula": "clamp(prev.position + velocity * dt, 0, 0.5)", 
      "units": "m",
      "state": true
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

Output cells declare their hardware target:

```json
{
  "pwmCommand": {
    "formula": "clamp(pid.output, 0, 100)",
    "output": "hardware",
    "hw": { "channel": "pwm0", "range": [0, 4095] }
  }
}
```

Input cells declare their hardware source:

```json
{
  "encoderCount": {
    "input": "hardware",
    "type": "int",
    "hw": { "channel": "encoder0", "countsPerRev": 4096 }
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

1. **Schema** - Document conforms to JSON and cell schema
2. **References** - All cell references resolve to existing cells
3. **Cycles** - No circular dependencies exist
4. **Types** - Formula return types are compatible with cell type
5. **Exclusivity** - Each cell has exactly one of: `value`, `formula`, `input`

A conforming implementation SHOULD validate:

1. **Units** - Dimensional consistency in formulas
2. **Ranges** - Initial values within declared ranges
3. **Reachability** - Warn on cells that are never referenced

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
    "rawAdc": { "input": "hardware", "type": "int", "range": [0, 4095] },
    "calibration": { "value": 0.245, "units": "lbs/count" },
    "tare": { "value": 892, "units": "count" },
    "force": { 
      "formula": "max(0, (rawAdc - tare) * calibration)", 
      "units": "lbs" 
    }
  },

  "controller": {
    "setpoint": { "input": "user", "value": 0, "units": "lbs", "range": [0, 800] },
    "feedback": { "formula": "loadCell.force", "units": "lbs" },
    
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
    "rawOutput": { 
      "formula": "kp * error + ki * integral + kd * derivative" 
    },
    "output": { 
      "formula": "safety.enabled ? clamp(rawOutput, -100, 100) : 0",
      "output": "hardware"
    }
  },

  "actuator": {
    "position": { "input": "hardware", "type": "float", "units": "mm" },
    "velocity": { 
      "formula": "dt > 0 ? (position - prev.position) / dt : 0", 
      "units": "mm/s",
      "state": true
    },
    "command": { "formula": "controller.output", "output": "hardware" }
  },

  "safety": {
    "maxForce": { "value": 800, "units": "lbs" },
    "maxPosition": { "value": 450, "units": "mm" },
    "maxVelocity": { "value": 100, "units": "mm/s" },
    
    "overload": { "formula": "loadCell.force > maxForce" },
    "overtravel": { "formula": "actuator.position > maxPosition" },
    "overspeed": { "formula": "abs(actuator.velocity) > maxVelocity" },
    
    "fault": { "formula": "overload || overtravel || overspeed" },
    "faultLatched": { 
      "formula": "fault || prev.faultLatched", 
      "state": true,
      "output": "indicator"
    },
    "enabled": { 
      "formula": "!faultLatched", 
      "output": "indicator" 
    },
    "reset": { "input": "user", "type": "bool", "value": false }
  },

  "sim_physics": {
    "_active": { "formula": "_config.simMode" },
    "mass": { "value": 5, "units": "kg" },
    "motorK": { "value": 2, "units": "N/%" },
    "damping": { "value": 50, "units": "N/(m/s)" },
    
    "motorForce": { "formula": "actuator.command * motorK", "units": "N" },
    "dampingForce": { "formula": "-damping * velocity", "units": "N" },
    "acceleration": { "formula": "(motorForce + dampingForce) / mass", "units": "m/s²" },
    "velocity": { 
      "formula": "prev.velocity + acceleration * dt", 
      "state": true, 
      "units": "m/s" 
    },
    "position": { 
      "formula": "clamp(prev.position + velocity * dt * 1000, 0, 500)", 
      "state": true, 
      "units": "mm" 
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
