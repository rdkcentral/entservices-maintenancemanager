# OpenSpec Directory Guide

This directory is the OpenSpec source of truth for architecture, lifecycle, and change management in this repository.

## Directory Layout

- openspec/config.yaml
  - OpenSpec schema/configuration.

- openspec/project.md
  - Canonical baseline architecture for MaintenanceManager.

- openspec/runtime/
  - Plugin lifecycle and runtime flow details.

- openspec/interfaces/
  - Northbound and southbound interface architecture.

- openspec/subsystems/
  - Deep dives into orchestration, state, and persistence.

- openspec/diagrams/
  - Mermaid diagrams for architecture and runtime sequences.

- openspec/unknowns/
  - Manual validation backlog and unresolved areas.

- openspec/changes/
  - Active and archived change proposals and implementation tasks.

- openspec/specs/
  - Canonical spec set used for long-term behavior definition.

## How to Use OpenSpec in Daily Development

## 1) Before starting work

- Read openspec/project.md and relevant subsystem/interface docs.
- Check active changes under openspec/changes/.
- If no suitable change exists, create one with the propose workflow.

## 2) During design and planning

- Record intended behavior in proposal/design/spec artifacts.
- Keep architecture assumptions explicit.
- Link implementation tasks to concrete acceptance criteria.

## 3) During implementation

- Execute tasks incrementally.
- Keep task progress updated in tasks.md.
- Document deviations from design with rationale.

## 4) During validation

- Validate against defined behavior, not assumptions.
- Add unresolved runtime questions to openspec/unknowns/.
- Capture evidence for behavior that depends on device/runtime context.

## 5) On completion

- Ensure code, docs, and OpenSpec artifacts agree.
- Archive completed changes using OpenSpec archive workflow.

## OpenSpec and Repository Documentation Sync Rules

- If code changes behavior, update OpenSpec and repo docs in the same effort.
- If docs disagree with code, code-truth should be documented and discrepancies called out.
- If behavior is uncertain, mark as unknown and add manual validation steps.

## Suggested Reading Order for New Contributors

1. openspec/project.md
2. openspec/runtime/plugin-lifecycle.md
3. openspec/runtime/maintenance-execution-flow.md
4. openspec/interfaces/jsonrpc-interface.md
5. openspec/interfaces/iarm-integration.md
6. openspec/subsystems/task-orchestration.md
7. openspec/subsystems/state-and-persistence.md
8. openspec/unknowns/manual-validation.md
