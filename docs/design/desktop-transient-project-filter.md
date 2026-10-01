# Desktop transient project scope

Status: implementation specification

Issue: [#1585](https://github.com/getagentseal/codeburn/issues/1585)

## Product decision

CodeBurn Desktop will add a searchable, single-project selector beside the
provider selector in the top bar. It is a temporary report scope for the
current app session, not a replacement for the persistent Projects settings.

`All projects` means the population already permitted by the Settings
include/exclude filter. Selecting a project narrows that population to one
canonical project identity. The Desktop app must apply that same narrowed
population before every report aggregates its data.

This is a Desktop-only feature. Electron will use a generic exact-project
argument when it invokes the local CodeBurn CLI, but that argument is internal
Desktop plumbing. It is not a new documented or supported end-user CLI
workflow.

## Goals

- Let a developer inspect exactly one project without changing persistent
  Settings visibility.
- Apply the selection consistently to Overview, Sessions, Pull Requests,
  Spend, Models, Optimize, Compare, and Compare Periods.
- Combine project, provider, period, and custom-date filtering by intersection.
- Preserve the selected project while navigating supported sections, changing
  periods/providers, and using Back/Forward.
- Reset to `All projects` on an app restart.
- Never render an unscoped or differently scoped cache entry under a selected
  project label.
- Keep hidden projects unavailable to the selector and avoid changing Settings
  as a side effect of temporary scoping.

## Non-goals

- Persisting a quick scope across app restarts.
- Selecting multiple projects, a project subtree, or a loose name match.
- Changing the Settings Projects include/exclude filter.
- Adding the selector to Plans, Plugins, or Settings.
- Changing exports or making a project selection a user-facing CLI feature.
- Filtering Combined-device payloads by a local project identity.

## Terms

### Persistent visibility filter

The existing Settings Projects include/exclude configuration. It is stored by
the Electron main process and becomes `--project` and `--exclude` CLI
arguments. It determines the outer population for every supported report.

### Quick project scope

The new session-only top-bar selection. Its value is either `all` or one
canonical `projectId`. It is never written through `setProjectFilter` and never
stored in localStorage.

### Canonical project identity

The exact identity produced by the existing `spendProjectIdentity` rule:

- Use the normalized absolute `projectPath` when one is available.
- Otherwise use the source project label.

This identity distinguishes projects that share a display name. It is the value
used for selection, IPC, CLI transport, filter composition, and cache keys.
Display labels are not identities.

### Effective device scope

The device scope actually used for a query. A quick project scope requires
local data. If the user's saved preference is Combined, the effective scope is
temporarily Local while a project is selected, without overwriting the saved
preference.

## User interaction

### Availability

`TopBar` renders the selector on these report sections:

- Overview
- Sessions
- Pull Requests
- Spend
- Models
- Optimize
- Compare
- Compare Periods

Plans, Plugins, and Settings do not render it. The selector is placed beside
the provider selector so project, provider, and date scope remain visible as
one report-control group.

### Picker behavior

The picker contains:

1. `All projects` as the first item.
2. One row per project permitted by the persistent visibility filter.

Each project row contains a canonical `id`, a display name, and its recorded
path. The collapsed control identifies the active project. The popup always
shows enough path to distinguish duplicate names; a path is also exposed in the
accessible name and tooltip when truncation is necessary.

Search matches display name and path to help the user find a project. Search
does not determine report membership. Selecting a row always sends its exact
canonical ID.

The picker is loaded lazily when it is first opened and is cached for the
session by the current persistent-filter revision. It has explicit loading,
empty, and retryable-error states:

- Loading leaves the current report scope unchanged.
- No visible projects leaves `All projects` selected and explains that Settings
  currently permits no project.
- A load error disables new selection for that open attempt but never broadens
  an already selected scope.
- A selected project with no data in the current provider/date slice remains
  selected and shows the ordinary empty state.

The popup follows the existing `Dropdown` accessibility conventions: a labeled
trigger, keyboard list navigation, Enter/Space selection, Escape dismissal,
and focus restoration to the trigger. It must remain usable at the compact
top-bar breakpoints; the visible label may compact, but the full active name and
path remain available to assistive technology.

### Clearing and Settings changes

Choosing `All projects` clears only the quick project scope. Reports immediately
return to the Settings-defined population.

When the persistent filter changes, the Desktop app invalidates the visible
project catalog, advances the report-filter generation, and revalidates the
active canonical ID. Requests and cached payloads from an earlier filter
generation cannot paint after that change. If the active project is no longer
eligible, the app clears the quick scope before issuing the replacement request.
The replacement still carries the persistent filter, so it cannot surface data
that the updated Settings policy hides.

## State lifetime and navigation

The renderer owns a separate app-level state:

```ts
type QuickProjectScope =
  | { kind: 'all' }
  | { kind: 'project'; id: string; label: string; path: string }
```

It is not an `InvestigationFilters` value and is not part of the Sessions
drill-through mechanism. It is also deliberately not stored in `NavState` or
the restart snapshot:

- It survives section changes, period/provider changes, custom ranges, and
  Back/Forward because those operations leave the app-level scope unchanged.
- Selecting or clearing a project refreshes the current report but does not add
  a Back/Forward history position.
- Restarting the app initializes the scope to `{ kind: 'all' }` even if scoped
  cache entries remain on disk.

This keeps the acceptance requirement that Back/Forward retain the current
quick scope while ensuring the scope cannot accidentally survive a restart.

## Data and filter contract

### Picker catalog

Add a typed bridge response for the top-bar picker:

```ts
type ProjectScopeOption = {
  id: string
  name: string
  path: string
}
```

The main process obtains the lifetime project catalog through the existing
reporting path with the persistent Settings arguments applied. The CLI response
is extended additively with the canonical identity rather than recomputing that
identity in the renderer. `getUnfilteredProjects` stays unfiltered for the
Settings checklist; the new scoped-catalog bridge is the selector's source.

The catalog is intentionally lifetime-based. A project with no activity in the
current period must still be selectable, so the control does not disappear or
change identity as the user changes a date or provider.

### Internal exact-ID transport

Generalize the existing cohort-only exact project selection into an internal
generic report argument:

```text
--project-id=<canonical-id>
```

The Electron main process validates an ID as a nonempty string without NUL and
passes it in `--option=value` form. This preserves dash-leading canonical IDs as
data rather than letting them be parsed as flags. The renderer may send zero or
one ID; multi-project selection is not introduced by this work.

The parser/report layer applies filters in this order:

```text
all parsed projects
  -> persistent Settings --project/--exclude matcher
  -> exact canonical --project-id matcher, when selected
  -> report aggregation
```

The existing include list is an OR set, so the quick scope must never be
implemented by appending another `--project` pattern. That would widen the
Settings population instead of intersecting it. The exact matcher compares the
canonical identity for equality and operates only on the already Settings-
filtered projects.

The internal argument is accepted by every report-producing command path the
Desktop app invokes, including the resident `serve` allowlist. Its absence
preserves current behavior. It is not added to `docs/cli.md` as a promised
end-user feature.

### Query coverage

Every Desktop request that contributes to a supported report receives the quick
scope, including:

- Overview/status and Pull Requests data.
- Sessions and Sessions contributions.
- Spend flow, timeline, and branch spend.
- Models and audit data used by the Models surface.
- Optimize report, Optimize snapshot, and Yield data.
- Compare model lists, comparisons, and cohort requests.
- Compare Periods report and its session drill-down.

Filtering happens before aggregation. A section may not post-filter a rendered
aggregate or reuse the existing Sessions investigation filter as a shortcut.

### Compare Periods

The same canonical ID applies to both A and B ranges and to the drill-down for
either range. The durable history/coverage cross-check must also receive the
exact scope. If a retained historical cache entry lacks enough project identity
to filter safely, that portion is reported as unavailable/detail-only rather
than merged into a scoped total. An unscoped aggregate must never supplement a
project-scoped Compare Periods result.

## Device scope

Combined-device usage cannot be reliably attributed to a local canonical
project. With `All projects`, the existing Local/Combined behavior is unchanged.
With a quick project scope:

- Queries use effective Local scope.
- The UI clearly communicates that Combined is unavailable while a project is
  selected.
- The user's requested Local/Combined preference is retained in memory and
  persisted unchanged.
- Clearing the project restores the requested device scope when the current
  provider/config selection permits it; otherwise the existing Local fallback
  continues to apply.

Quick-project selection must not use or trigger the existing code path that
permanently writes a Local preference for a persistent project filter.

## Cache, refresh, and stale-response rules

Define a stable `projectScopeKey`: `all` for the unscoped state and an encoded
canonical ID for the selected state. Add it to every identity that can hold
report output:

- `overviewMemoKey` and overview headline snapshots.
- `reportMemoKey` for section reports, including Period Compare and drill-down
  variants.
- `selectedReportMemoKeys`, refresh timestamps, and polling dependencies.
- Warm/prefetch keys and the optimize snapshot key.
- Any in-flight request identity or stale-response guard.

Treat the current persistent Settings filter as a separate report-filter
generation. A changed include/exclude value invalidates report and headline
memos, invalidates the selector catalog, and makes in-flight results from the
previous generation stale. The request identity checks both that generation and
the quick-scope key before storing or rendering a result.

Changing the quick scope creates a new query identity. A previously resolved
payload may remain cached under its own identity but cannot paint beneath a
different scope label. The renderer must only write a payload or snapshot after
confirming its key still matches the request that produced it.

Existing broad prefetching is optional while a project is selected. If retained,
it must use the quick-scope key; it may not warm or reuse an unscoped response
for the selected project.

## Error handling

- Reject malformed exact IDs in the main process before constructing argv.
- Treat a missing or stale catalog entry as a catalog state, not as evidence that
  a valid selected project has no report data.
- Treat a report result with zero rows as a valid scoped result, not as a reason
  to clear the selection.
- Surface CLI/IPC failures through existing report error states. Do not fall back
  to an unfiltered request.
- Preserve the last exact scoped snapshot while that same scoped request
  refreshes; never preserve a different scope as a visual fallback.

## Code boundaries

- `src/parser.ts` and a shared canonical-project helper: compose persistent
  pattern filtering with exact canonical-ID filtering.
- `src/main.ts`, `src/serve.ts`, and period-diff/history code: accept the
  internal exact-ID argument on each Desktop report path and preserve scope in
  Compare Periods history and drill-down data.
- `app/electron/main.ts`: validate one transient ID, append it after persistent
  project arguments, provide the visible lifetime catalog, and include it in
  every supported bridge handler.
- Preload and `app/renderer/lib/types.ts`: expose the typed catalog and optional
  quick-scope argument on report methods.
- `app/renderer/App.tsx`: own session-only scope, derive effective device scope,
  invalidate/revalidate the catalog after Settings changes, and thread the scope
  into all report components and memo keys.
- `app/renderer/components/TopBar.tsx` plus a focused picker component: render
  the accessible project selector without altering unsupported sections.
- `app/renderer/lib/reportMemoKey.ts` and related overview-key helpers: make
  scope an explicit cache identity component.

## Verification

Automated coverage must establish all of the following:

- Canonical exact matching distinguishes duplicate names and does not include a
  path descendant or substring merely because its display text matches.
- Persistent Settings includes/excludes apply first; the transient ID narrows
  rather than widens the result.
- Every Electron bridge handler for a supported report receives the exact-ID
  argument, while Plans, Plugins, Settings, exports, and the Settings catalog do
  not acquire transient filtering.
- The CLI and resident `serve` paths reject malformed IDs safely and retain
  existing behavior when no ID is supplied.
- Overview, report, prefetch, optimize, and Compare Periods memo keys differ
  across `All projects` and each canonical ID.
- A late response for one scope cannot replace the data for another scope.
- A persistent Settings-filter revision invalidates its catalog and reports, and
  a response from the previous revision cannot render afterward.
- The quick scope survives section changes, provider/period/custom-range
  changes, and Back/Forward, but returns to `All projects` after a restart.
- Selecting a project while Combined is preferred runs locally without changing
  the saved preference; clearing restores Combined.
- A Settings mutation hiding the active project clears it, while a selected
  project with no current-period data remains selected and shows an empty state.
- The picker supports search, duplicate-name path disambiguation, keyboard
  selection, Escape, accessible naming, loading, empty, and error states.
- Compare Periods scopes both ranges, coverage/history, and its session
  drill-down to the same project corpus.

Relevant suites include `app/electron/main.test.ts`,
`app/renderer/App.test.tsx`, `app/renderer/sections/PeriodCompare.test.tsx`,
`app/renderer/lib/reportMemoKey.test.ts`, and
`tests/compare-periods-cli.test.ts`, plus new exact-project helper and bridge
contract tests.

## Decisions

| Decision | Rationale |
| --- | --- |
| The feature is Desktop-only. | Issue #1585 requests a desktop top-bar selector, not a new CLI workflow. |
| Exact-ID CLI transport is internal plumbing, not documented CLI product surface. | Electron invokes the CLI today; a generic internal argument gives every report one reliable backend contract without expanding the public CLI promise. |
| The picker permits `All projects` or exactly one canonical project. | This satisfies the user story and avoids accidental multi-project, subtree, or substring semantics. |
| Canonical identity comes from normalized absolute path when available, otherwise the source project label. | It matches existing Spend and cohort identity behavior and distinguishes duplicate names. |
| Persistent Settings filtering is the hard outer boundary. | Users must never see a project Settings has hidden. Exact selection is an intersection, not another include pattern. |
| Quick scope is app-session state outside `NavState`. | It stays active through navigation and Back/Forward, while restart reliably resets it. |
| The selector uses a visible, lifetime project catalog. | A project remains selectable even if the current period/provider has no data; hidden projects never appear. |
| Plans, Plugins, Settings, and exports remain unscoped. | They are not report surfaces covered by the issue, and transient Desktop scope must not alter persistent/export behavior. |
| A selected project forces effective Local device scope without changing the saved device preference. | Combined-device payloads cannot be reliably filtered by local canonical identity. |
| Every cache, snapshot, refresh, prefetch, and stale-response identity includes project scope. | This prevents data for another project or `All projects` from displaying under the wrong scope. |
| A persistent Settings-filter change starts a new report-filter generation. | Changing visibility policy must invalidate cached and in-flight data from the old population. |
| Compare Periods filters A, B, history/coverage, and drill-downs with the same ID. | Report totals must derive from one consistent filtered corpus. |
| A Settings change that hides the active project clears the quick scope; an empty result does not. | The former protects visibility policy, while the latter is a valid period/provider result. |
