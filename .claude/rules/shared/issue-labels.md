# Work Issue Labels

Every issue shall have a set of labels attached in the issue tracker.

## 0. Color reservation

**Red and orange hues are reserved for `priority:` labels.** Urgency is the only thing a reader
should perceive as "hot" at a glance. No `type:` or `scope:` label may use a red or orange tone
— those categories draw from the rest of the spectrum (yellow, green, teal, cyan, blue, indigo,
violet, magenta) plus neutral gray. Within each category the hues are spread far enough apart to
stay distinguishable side by side.

## 1. Issue Type (Type)

Defines the fundamental nature of the issue. Each issue must have exactly one `type:` label.

| Label               | Color (Hex) | Description                                                               |
| :------------------ | :---------- | :------------------------------------------------------------------------ |
| `type: bug`         | `#D6249F`   | Bugs, crashes, or unintended behavior in the existing codebase.           |
| `type: feature`     | `#2ECC71`   | Major new functionality or large requirements.                            |
| `type: enhancement` | `#3498DB`   | Minor improvements, polishing, or extensions of existing features.        |
| `type: suggestion`  | `#9B59B6`   | Ideas, feedback, or proposals that need discussion before implementation. |
| `type: study`       | `#F1C40F`   | Research, spikes, POCs (Proof of Concept), or technical investigations.   |
| `type: refactor`    | `#3F51B5`   | Code cleanup or architectural adjustments with no behavior changes.       |
| `type: chore`       | `#7F8C8D`   | Maintenance tasks, CI/CD, dependency upgrades.                            |
| `type: docs`        | `#1ABC9C`   | Documentation changes, API specs, README updates.                         |
| `type: perf`        | `#8E44AD`   | Performance optimizations (load times, bundle size, render speed).        |

## 2. Technology & Layer (Scope)

Identifies the affected layers and frameworks of the project. Multiple scope labels per issue are
permitted.

| Label             | Color (Hex) | Description                                                     |
| :---------------- | :---------- | :-------------------------------------------------------------- |
| `scope: frontend` | `#5DADE2`   | General UI/UX changes, CSS, visual components.                  |
| `scope: backend`  | `#6C3483`   | Server-side logic, API endpoints, runtime-specific features.    |
| `scope: types`    | `#B7950B`   | TypeScript-specific tasks, type definitions, strict-mode fixes. |
| `scope: database` | `#27AE60`   | Schema migrations, DB queries, ORM/driver layer.                |

## 3. Priority (Priority)

Controls execution urgency. Each active issue must have at most one `priority:` label.

| Label                | Color (Hex) | Description                                                             |
| :------------------- | :---------- | :---------------------------------------------------------------------- |
| `priority: critical` | `#FF0000`   | Production outage, data loss, or team blocker. Fix immediately.         |
| `priority: high`     | `#E74C3C`   | Critical for the current release/milestone. Blocks downstream features. |
| `priority: medium`   | `#F39C12`   | Standard priority for scheduled features and normal bugs.               |
| `priority: low`      | `#BDC3C7`   | Non-critical, cosmetic fixes, or nice-to-have enhancements.             |

## 4. No readiness labels

Readiness is never a label. An issue is ready to be worked on when everything in its
`## Blocked by` section is closed. Never create or apply `ready-for-agent`, `ready-for-human`,
`needs-triage` or any other triage-state label — also not when a skill asks for one.

## Agent integration instructions

When creating new issues or categorizing existing ones, strictly adhere to this matrix. Write the
labels exactly as above: the category, a colon, a space, the value. The issue tracker does not
enforce that `type:` and `priority:` are exclusive; this rule does.
