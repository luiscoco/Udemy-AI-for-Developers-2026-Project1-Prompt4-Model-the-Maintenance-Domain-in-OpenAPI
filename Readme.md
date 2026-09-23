# Model the Maintenance Domain in OpenAPI

This step models the **data contract** for the Equipment Maintenance Hub before any
route or implementation code is written. The goal is to describe the domain
(work orders, assets, technicians) as reusable OpenAPI schemas, so both the
frontend (React/Vite) and backend (Fastify) teams can code against the same
shapes.

## What was built

A single file, [`packages/contract/openapi.yaml`](packages/contract/openapi.yaml),
containing only `components/schemas` — no `paths` yet. That comes in a later
prompt once the domain model is agreed on.

## Steps followed

1. **Started from the lifecycle, not the fields.**
   The prompt described a work order lifecycle as a state diagram:

   ```
   reported --triage--> triaged --schedule--> scheduled --start--> in_progress --complete--> completed
                            |                    |
                            +--cancel----------> cancelled
                                                 ^
                            scheduled --cancel---+
   ```

   This became two enums first, before any object schema:
   - `WorkOrderState` — the six possible states a work order can be in.
   - `WorkOrderAction` — the five commands that move a work order between
     states (`triage`, `schedule`, `start`, `complete`, `cancel`).

   Modeling the state machine as data (rather than hardcoding transitions in
   route logic) means the same enums can later validate both API requests and
   UI dropdowns.

2. **Defined supporting enums and entities.**
   - `Priority` (`low` / `medium` / `high` / `critical`) — a simple enum, since
     priority doesn't have transition rules like state does.
   - `Asset` and `Technician` — the two entities a work order references
     (`assetId`, `technicianId`), each with a minimal identifying set of
     fields.

3. **Split "read" vs "write" shapes for WorkOrder.**
   Rather than one `WorkOrder` schema reused everywhere, two schemas were
   created:
   - `WorkOrder` — the full resource as returned by the API, including
     server-generated fields (`id`, `reference`, `state`, `reportedAt`,
     `updatedAt`).
   - `NewWorkOrder` — only the fields a client supplies when reporting a new
     work order (`assetId`, `title`, `description`, `priority`). It
     deliberately excludes `state` and `technicianId`, since a new work order
     always starts at `reported` with no technician, matching the lifecycle
     diagram.

   This separation avoids a client being able to set fields like `state` or
   `id` directly on creation — those are server-owned.

4. **Modeled state changes and assignment as commands, not field edits.**
   - `TransitionCommand` — wraps a single `action` (from `WorkOrderAction`).
     Sent to whatever endpoint later performs a lifecycle transition.
   - `AssignmentCommand` — wraps a `technicianId`. Sent to whatever endpoint
     later assigns a technician.

   This keeps the "how do I change a work order" question explicit and
   auditable, instead of allowing arbitrary `PATCH` of the `state` field
   (which would let a client skip straight from `reported` to `completed`,
   bypassing the lifecycle rules).

5. **Added a `DashboardSummary` for aggregate/reporting needs.**
   Modeled as counts (`totalOpen`, `criticalOpen`, `unassigned`) plus a
   `byState` breakdown, represented as an open map
   (`additionalProperties: integer`) rather than one property per state —
   so the schema doesn't need to change if `WorkOrderState` grows.

6. **Added a standard `ApiError` shape.**
   A `message` string (required) plus an optional, open-ended `details`
   object, so every endpoint can return errors in a consistent shape without
   over-specifying error payloads this early.

7. **Marked required fields explicitly on every object schema**, using
   OpenAPI's `required` array, rather than relying on defaults. This makes it
   unambiguous which fields a client must send (or the server always
   returns) versus which are optional/nullable.

8. **Left `paths` empty on purpose.**
   The prompt asked for schemas only. Defining endpoints before the domain
   model is settled tends to lock in premature decisions (URL shapes, verbs,
   status codes) before the team has agreed on what a "work order" even
   looks like.

9. **Called out design assumptions instead of silently deciding them.**
   Anywhere the prompt was ambiguous (ID format, error `details` shape,
   whether counts can be negative, timestamp format), a judgment call was
   made and then listed explicitly, so the team can confirm or override it
   rather than discover it later by reading the YAML closely.

## Design assumptions to review

- IDs (`id`, `assetId`, `technicianId`) are typed as `string` with
  `format: uuid`. Change this if the real ID scheme differs.
- `reference` is a free-text `string` with no enforced pattern (e.g.
  `WO-2026-00042`).
- `ApiError.details` is an open, untyped object (`additionalProperties: true`).
- `DashboardSummary` counts have no `minimum: 0` constraint yet.
- Timestamps use ISO 8601 `date-time` format.
- `NewWorkOrder` intentionally omits `technicianId` and `state` — assignment
  and transitions happen through their own commands, not at creation time.

## Running / previewing this step (Windows terminal)

There is no runnable application yet — `apps/frontend` and `apps/backend`
are still empty skeletons (just a `package.json` each), and the root
`npm run dev` script is a placeholder. The only artifact this step
produces is the OpenAPI contract file itself, so "running" it means
previewing/validating the YAML rather than starting a server.

From the repository root, in PowerShell or Command Prompt:

```powershell
npx @redocly/cli preview-docs packages/contract/openapi.yaml
```

This starts a local docs server (default `http://localhost:8080`) that
renders `openapi.yaml` as interactive API documentation, and reloads on
save. No install step is required — `npx` downloads the CLI on first run.

To just validate the file is well-formed OpenAPI without opening a server:

```powershell
npx @redocly/cli lint packages/contract/openapi.yaml
```

Once `apps/frontend` and `apps/backend` have real source code and `dev`
scripts (a later prompt), this section should be updated with the actual
`npm run dev` command to start the app.

## Suggested next steps

- Review the assumptions above as a team and amend the schemas if any are
  wrong.
- Add `paths` (endpoints) that reference these schemas, e.g.
  `POST /work-orders` using `NewWorkOrder`, or
  `POST /work-orders/{id}/transitions` using `TransitionCommand`.
- Consider adding validation rules (`minLength`, `pattern`, `minimum`) once
  the endpoints make those constraints concrete.
