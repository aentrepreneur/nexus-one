# NEXUS ONE Architecture

## Purpose

NEXUS ONE turns an existing project into a portable autonomous agent harness.

## Public Architecture Layers

```text
Source Project
  -> Scanner
  -> Generator
  -> Validation Layer
  -> Portable Harness
  -> VPS Runtime
```

## Main Components

- Scanner: inspects structure, runtime, and entry points
- Generator: creates portable harness files
- Validation layer: checks integrity and packaging rules
- Runtime layer: bootstraps the generated harness in target environments

## Execution Model

### Development Side

- source project inspection
- packaging decisions
- harness generation
- validation before deployment

### Runtime Side

- target environment detection
- integrity verification
- controlled bootstrapping
- constrained operational execution

## Design Notes

- The public repo documents the framework model, not sensitive implementation details.
- Generated harnesses are treated as deployable operational artifacts.
- The separation between generator and runtime is part of the core design.
