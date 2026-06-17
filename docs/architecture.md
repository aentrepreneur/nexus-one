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

## Design Notes

- The public repo documents the framework model, not sensitive implementation details.
- Generated harnesses are treated as deployable operational artifacts.
