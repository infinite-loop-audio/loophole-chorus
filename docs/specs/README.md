# Specifications

This folder contains **machine-friendly specifications** that define how the
Loophole components communicate and structure data.

Typical contents include:

- IPC message schemas between:
  - Aura ↔ Signal
  - Aura ↔ Pulse (when Pulse becomes a separate service)
- Engine command formats
- Telemetry packet formats
- Plugin descriptor and parameter schemas

All specs should be written in formats that are easy for tools to consume:

- JSON Schema
- TypeScript interfaces
- C-style structs
- Small Markdown files with clearly marked sections

Implementation in other repositories (Signal, Pulse, Aura) must follow these
specifications. Changes to specs should be driven via the meta-protocol
defined in `@chorus:/docs/meta/`.
