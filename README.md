# NEXUS ONE

Harness Engineering Framework for portable autonomous agents.

## Overview

NEXUS ONE is a framework for turning existing projects into deployable autonomous agent harnesses.

Instead of treating an agent as only a prompt or chat wrapper, NEXUS ONE treats it as an operational system with packaging, integrity, runtime detection, deployment logic, and execution constraints.

This repository is intentionally public-safe. It explains the framework model, deployment logic, and system value without exposing internal operational details that should remain private.

## What It Does

- Scans an existing project and derives a deployable harness
- Generates a portable runtime structure for VPS environments
- Preserves source context while separating generated operational layers
- Supports validation, dry-runs, and repeatable packaging
- Enables autonomous monitoring, remediation, and operator-assist workflows

## Why Harness Engineering

NEXUS ONE is built around a simple idea: autonomous systems should be deployable like operational products, not treated as disposable prompt wrappers.

Harness Engineering creates a bridge between:

- an existing codebase or operational project
- a generated portable harness
- a controlled runtime in a target environment

That makes the resulting system easier to reason about, validate, package, and operate.

## Core Model

### Level 1: Generator

The framework analyzes a project and produces a self-contained harness.

### Level 2: Generated Harness

The generated harness is the deployable unit that runs in a VPS or controlled environment.

### Operational Boundary

The generator and the generated harness are intentionally different concerns:

- the generator exists in the development environment
- the generated harness exists in the runtime environment

This separation helps keep deployment logic portable and operationally focused.

## Public Structure

```text
nexus-one/
  prod-agent/
    prod-agent.sh
    modules/
    templates/
  remediation/
  docs/
```

Generated output pattern:

```text
ag-project/
  boot.sh
  detect.sh
  .agent/
  src/
  integrity.sum
```

See `docs/architecture.md` for the public architecture view and `examples/` for safe layout examples.

## Example Workflow

```bash
./prod-agent.sh --scan /opt/my-project
./prod-agent.sh --scan --dry-run /opt/my-project
./prod-agent.sh --validate /opt/ag-my-project
```

## Public Use Cases

- package an existing project into a deployable autonomous harness
- create portable monitoring and remediation runtimes
- establish a repeatable deployment model for agentic systems
- support human-in-the-loop operational agents in controlled environments

## Why It Matters

NEXUS ONE is an agentic engineering system, not just an agent launcher. It formalizes packaging, deployability, integrity, and runtime behavior so autonomous systems can be treated as real operational components.

## Repository Structure

```text
nexus-one/
  README.md
  docs/
    architecture.md
    design-principles.md
    roadmap.md
    use-cases.md
  examples/
    commands.md
    harness-layout.txt
    lifecycle.txt
```

## Documentation

- `docs/architecture.md`: public architecture and execution layers
- `docs/design-principles.md`: framework intent and design rules
- `docs/use-cases.md`: scenarios and environments
- `docs/roadmap.md`: public direction and maturity path

## Use Cases

- Portable monitoring agents
- Autonomous remediation layers
- Controlled operator-assist systems
- VPS-deployable agent runtimes
- Security and operations automation loops

## Status

Public-facing architecture brief. Internal implementation remains private or selectively published.
