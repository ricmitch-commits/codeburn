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

This is a Desktop-only feature. Electron will use a hidden exact-project
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
- Changing exports or making a project selection or its transport a user-facing
  CLI feature.
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

When distinct paths are known, this identity distinguishes projects that share
a display name. It is the value used for selection, IPC, CLI transport, filter
composition, and cache keys. Display labels are not identities except for the
pathless fallback where source data supplies no stronger identity.

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
path when known. The collapsed control identifies the active project. The popup
always shows enough path to distinguish duplicate names when path data exists;
a path is also exposed in the accessible name and tooltip when truncation is
necessary. A pathless entry displays `Path unavailable`. Two pathless records
with the same source label are one canonical identity because the source data
cannot distinguish them, so the picker does not claim they are separately
selectable.

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
  | { kind: 'project'; id: string; label: string; path: string | null }
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
  path: string | null
}
```

The main process obtains the catalog through a hidden
`report --format desktop-project-catalog --period lifetime` contract with the
persistent Settings arguments applied. This is not the existing public JSON
report: that report lists only live projects, while a scope picker must also
know safely attributable retained history. The internal catalog unions canonical
identities from live parsed projects and exact-ID daily-cache records. It omits
legacy cache records that cannot safely identify one canonical project.

The CLI creates the canonical identity rather than recomputing it in the
renderer. `getUnfilteredProjects` stays unfiltered for the Settings checklist;
the new scoped-catalog bridge is the selector's source.

The catalog is intentionally lifetime-based. A project with no activity in the
current period must still be selectable, so the control does not disappear or
change identity as the user changes a date or provider.

### Internal exact-ID transport

Add a hidden, Desktop-only report argument:

```text
--desktop-project-id=<canonical-id>
```

The Electron main process validates an ID as a nonempty string without NUL and
passes it in `--option=value` form. This preserves dash-leading canonical IDs as
data rather than letting them be parsed as flags. The renderer may send zero or
one ID; multi-project selection is not introduced by this work.

`--desktop-project-id` is deliberately distinct from the existing documented,
repeatable, cohort-only `compare --format cohort-json --project-id` option. The
existing cohort option retains its current behavior. When a cohort report has
both options, the Desktop scope is applied first and the cohort selection
narrows that result further; the two options never form a union. The hidden
Desktop option is absent from public CLI help and documentation.

The parser/report layer applies filters in this order:

```text
all parsed projects
  -> persistent Settings --project/--exclude matcher
  -> exact canonical --desktop-project-id matcher, when selected
  -> command-specific filters, including cohort --project-id when present
  -> report aggregation
```

The existing include list is an OR set, so the quick scope must never be
implemented by appending another `--project` pattern. That would widen the
Settings population instead of intersecting it. The exact matcher compares the
canonical identity for equality and operates only on the already Settings-
filtered projects.

The internal argument is accepted by every report-producing command path the
Desktop app invokes, including the resident `serve` allowlist. Its absence
preserves current behavior. It is not added to `docs/cli.md` or exposed through
public help as a promised end-user feature.

Core CLI validation and the resident `serve` allowlist both reject
`--desktop-project-id` combined with `--scope combined`. Electron derives Local
before building the request, but the lower-layer rejection prevents an older or
stale renderer from accidentally requesting an unfilterable Combined payload.

### Typed Desktop report query

Do not append another ambiguous positional argument to every bridge method.
Introduce one named typed options object for Desktop report calls. It carries
the optional range, background priority, effective device scope, and quick
project ID together:

```ts
type DesktopReportQuery = {
  range?: DateRange
  background?: boolean
  deviceScope?: 'local' | 'combined'
  projectId?: string
}
```

The renderer computes effective Local scope before creating this object. The
preload and main process validate it once, and every supported bridge method
consumes the same object. Renderer-only filter revision remains part of request
and memo identity, but is never sent to the CLI.

### Query coverage

Every Desktop request that can accurately contribute to a supported report
receives the quick scope, including:

- Overview/status and Pull Requests data.
- Sessions and Sessions contributions.
- Spend flow, timeline, and branch spend.
- Models and audit data used by the Models surface.
- Optimize report, Optimize snapshot, and Yield data.
- Compare model lists, comparisons, and cohort requests.
- Compare Periods report and its session drill-down.

Filtering happens before aggregation. A section may not post-filter a rendered
aggregate or reuse the existing Sessions investigation filter as a shortcut.

### Complete filter intersection

Provider and custom-range changes must continue to intersect with a quick
project scope on every supported surface. Existing gaps are in scope for this
work, not exceptions to the contract:

- Classic Compare gains `--from`/`--to` support and receives the selected custom
  range instead of displaying its current range-warning fallback.
- Yield gains `--from`/`--to` support; the bridge forwards the selected custom
  range rather than dropping it.
- Every supported report, including its prefetch and memo/poll dependencies,
  receives the same period or custom range alongside the provider and project
  scope.

Pull Requests must aggregate its rows from the already provider- and
project-filtered `scanProjects` corpus. Remove the all-provider-only omission
for local scoped reports; a selected provider must produce its matching PR data
or an honest empty result, never an unfiltered aggregate or an artificial empty
state caused by the provider gate.

### Scoped Optimize

Optimize findings must have an explicit provenance policy. A detector can render
under a quick project scope only when it derives its evidence from the selected
canonical project corpus. `scanSessions` and other file scanners receive the
selected canonical identities/paths before discovery or read only files proven
to belong to them. Global configuration detectors (for example, user-wide MCP,
skill, command, or tool configuration) are omitted while scoped unless they can
prove attribution to the selected project.

The Optimize result-cache identity includes `projectScopeKey`. A scoped Optimize
result may never reuse a global result that happens to have the same aggregate
counts. A finding that combines project evidence with global configuration is
also omitted until the configuration evidence can be attributed safely.

### Unattributable applied actions

`act report` and the applied-fix data included by `optimize` are currently
global journal data; they do not accept or retain a canonical project identity.
They cannot truthfully appear in a project-scoped report. While a quick project
scope is active:

- Overview does not request or render the `Saved by applied fixes` line.
- Optimize omits its global applied-action header and applied-fix rows.
- Existing global applied-action memo entries are not reused under the project
  label.

Scoped action reporting is future work only after actions have trustworthy
canonical project attribution. Hiding it is preferable to presenting a global
value as a project value.

### Durable historical identity

The daily cache must preserve the same canonical project identity used by live
reports. Its per-project day maps, including provider slices, hold structured
project buckets with all of the following data:

- canonical ID;
- raw source label used by the Settings matcher;
- display label;
- optional recorded path; and
- provenance: `exact` or `legacy`.

Exact bucket map keys are namespaced encodings of canonical IDs. Legacy bucket
keys use a different namespace. Cache merge, migration, and provider-overlay
code compare those bucket keys and provenance; they never merge an old
label-keyed bucket into an exact bucket merely because the strings match.

Persistent Settings matching on a retained bucket uses its preserved raw label
and path, matching `makeProjectFilter` semantics. A quick scope then requires
`provenance: 'exact'` and canonical-ID equality.

This is a cache-schema change. Bump the daily-cache version and the status
snapshot semantic/render identity, then rederive data where source sessions are
available. Historical cache rows written under the old label-keyed schema may
already have combined same-named projects. Migrate them to the legacy namespace
without claiming an exact identity. They remain usable for unscoped totals, but
are marked legacy/unattributable and excluded from:

- quick-project report totals;
- quick-project history and coverage calculations; and
- the quick-project catalog.

The status snapshot `queryScope` also includes `desktopProjectId`, so a disk
snapshot for one project cannot satisfy another project's status request.

### Compare Periods

The same canonical ID applies to both A and B ranges and to the drill-down for
either range. The durable history/coverage cross-check uses only exact-ID daily
cache records. If retained history is legacy/unattributable, that portion is
reported as unavailable/detail-only rather than merged into a scoped total. An
unscoped aggregate must never supplement a project-scoped Compare Periods
result.

### Scoped Overview payload audit

Every field rendered by Overview has one explicit scoped policy:

| Payload/data | Policy while quick-scoped |
| --- | --- |
| Current totals, sessions, models, spend, and daily history | Recompute from the exact selected corpus and exact retained buckets only. |
| Pull Requests and branches | Aggregate from the same provider- and exact-project-filtered live corpus. |
| `periodTotals` | Omit. It is an unscoped warm-generation optimization and must not stand in for a project answer. |
| Streak | Recompute from exact selected day buckets; show unavailable rather than read raw machine-wide cache days. |
| `act report` / realized applied savings | Omit, as defined below. |
| Optimize findings | Include only findings passing the Scoped Optimize provenance policy. |

`hasProjectFilter` and equivalent payload gates must treat a
`desktopProjectId` as a project filter. No global generation field is retained
merely because the persistent Settings filter itself is empty.

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

Define a collision-free `projectScopeKey`: exactly `all` for the unscoped state
and `project:<base64url(UTF-8 canonical-id)>` for a selected project. Add it to
every identity that can hold report output:

- `overviewMemoKey` and overview headline snapshots.
- `reportMemoKey` for section reports, including Period Compare and drill-down
  variants.
- `selectedReportMemoKeys`, refresh timestamps, and polling dependencies.
- Warm/prefetch keys and the optimize snapshot key.
- The CLI status snapshot `queryScope` and its semantic cache identity.
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
  pattern filtering with exact canonical-ID filtering without changing public
  cohort `--project-id` semantics.
- `src/day-aggregator.ts`, `src/daily-cache.ts`, and status-snapshot code: key
  per-project retained history by structured canonical buckets, preserve raw
  Settings-match metadata and provenance, bump cache/snapshot versions, and
  exclude legacy ambiguous rows from quick scopes.
- `src/main.ts`, `src/serve.ts`, and period-diff/history code: accept the hidden
  internal `--desktop-project-id` on each Desktop report path, provide the
  hidden lifetime catalog, add Compare/Yield custom-range support, make Pull
  Requests provider-aware, and preserve scope in Compare Periods history and
  drill-down data.
- `src/optimize.ts`: restrict or omit every detector by exact project
  provenance, and include quick scope in its result-cache identity.
- `app/electron/main.ts`: validate one transient ID, build a typed
  `DesktopReportQuery`, append it after persistent project arguments, provide
  the visible lifetime catalog, and include it in every supported bridge handler.
- Preload and `app/renderer/lib/types.ts`: expose the typed catalog and named
  `DesktopReportQuery` rather than adding positional scope parameters.
- `app/renderer/App.tsx`: own session-only scope, derive effective device scope,
  invalidate/revalidate the catalog after Settings changes, and thread the scope
  into all report components and memo keys.
- `app/renderer/components/TopBar.tsx` plus a focused picker component: render
  the accessible project selector without altering unsupported sections.
- Overview and Optimize components: omit global applied-action data whenever a
  quick project scope is active.
- `app/renderer/lib/reportMemoKey.ts` and related overview-key helpers: make
  scope an explicit cache identity component.

## Verification

Automated coverage must establish all of the following:

- Canonical exact matching distinguishes duplicate names and does not include a
  path descendant or substring merely because its display text matches.
- Persistent Settings includes/excludes apply first; the transient ID narrows
  rather than widens the result.
- The documented cohort `--project-id` remains repeatable and unchanged; a
  hidden Desktop ID intersects with it rather than widening its population.
- Every Electron bridge handler for a supported report receives the exact-ID
  argument, while Plans, Plugins, Settings, exports, and the Settings catalog do
  not acquire transient filtering.
- The hidden CLI and resident `serve` paths reject malformed IDs safely, are
  absent from public help, and retain existing behavior when no ID is supplied.
- Both CLI and `serve` reject `--desktop-project-id` with Combined scope, even
  when a stale caller bypasses the Desktop effective-scope calculation.
- Bridge calls use the named `DesktopReportQuery` contract so range, priority,
  device scope, and project ID cannot shift into one another positionally.
- The lifetime picker catalog unites live projects and safely attributable
  retained identities, while legacy ambiguous and pathless duplicate data is
  never presented as separately selectable.
- Daily-cache project maps and status snapshot identities use canonical IDs;
  exact buckets retain raw Settings-match label/path metadata and provenance;
  legacy label-keyed entries cannot contribute to a quick-project total or
  merge into an exact bucket.
- Overview, report, prefetch, optimize, and Compare Periods memo keys differ
  across `All projects` and each canonical ID.
- Project-scope keys are collision-free for the unscoped sentinel and any valid
  canonical ID.
- A late response for one scope cannot replace the data for another scope.
- A persistent Settings-filter revision invalidates its catalog and reports, and
  a response from the previous revision cannot render afterward.
- The quick scope survives section changes, provider/period/custom-range
  changes, and Back/Forward, but returns to `All projects` after a restart.
- Selecting a project while Combined is preferred runs locally without changing
  the saved preference; clearing restores Combined.
- A Settings mutation hiding the active project clears it, while a selected
  project with no current-period data remains selected and shows an empty state.
- Scoped Overview and Optimize do not render global applied-action savings,
  headers, or applied-fix rows.
- Scoped Overview omits `periodTotals` and computes streak only from exact
  selected buckets, reporting unavailable where retained history is unsafe.
- Scoped Optimize omits unprovable global configuration findings, scopes its
  file scans and result cache, and returns only selected-project findings.
- Classic Compare and Yield honor the selected custom range; all supported
  reports intersect provider, period/range, and project scope.
- Pull Requests return a provider-scoped, exact-project aggregation rather than
  relying on the current all-provider-only payload path.
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
| Hidden `--desktop-project-id` transport is internal plumbing, not documented CLI product surface. | Electron invokes the CLI today; a distinct hidden argument gives every report one reliable backend contract without changing the documented cohort `--project-id` promise. |
| The picker permits `All projects` or exactly one canonical project. | This satisfies the user story and avoids accidental multi-project, subtree, or substring semantics. |
| Canonical identity comes from normalized absolute path when available, otherwise the source project label. | It matches existing Spend and cohort identity behavior and distinguishes duplicate names when source paths are known. |
| Persistent Settings filtering is the hard outer boundary. | Users must never see a project Settings has hidden. Exact selection is an intersection, not another include pattern. |
| Desktop bridge report calls use one named query object. | Range, background priority, device scope, and project identity remain unambiguous as the bridge evolves. |
| Quick scope is app-session state outside `NavState`. | It stays active through navigation and Back/Forward, while restart reliably resets it. |
| The selector uses a visible, lifetime project catalog. | A project remains selectable even if the current period/provider has no data; hidden projects never appear. |
| The lifetime catalog merges live projects with safely attributable retained identities; pathless same-label records are one entry. | The picker remains useful for retained history without inventing a distinction the source data cannot prove. |
| Plans, Plugins, Settings, and exports remain unscoped. | They are not report surfaces covered by the issue, and transient Desktop scope must not alter persistent/export behavior. |
| A selected project forces effective Local device scope without changing the saved device preference. | Combined-device payloads cannot be reliably filtered by local canonical identity. |
| Daily-cache project buckets retain canonical ID, raw filter label, display data, path, and provenance, and receive a version bump. | Settings matching needs the raw label/path, while namespaced `exact` and `legacy` buckets prevent ambiguous retained data from merging into an exact scope. |
| Global applied-action data is hidden while scoped. | Current action journals lack exact project attribution, so showing them would mislabel global savings as one project's data. |
| Optimize reports only project-proven findings while scoped. | Global configuration scans and their cached results cannot be represented as one project's data without provenance. |
| Classic Compare, Yield, and Pull Requests receive missing range/provider propagation. | The issue requires every supported report to show the intersection of project, provider, and custom-date filters. |
| Scoped Overview fields follow the explicit payload audit. | Generation-only totals and unsafe global history must be omitted or recomputed, never relabeled as one project's data. |
| Every cache, snapshot, refresh, prefetch, and stale-response identity includes project scope. | This prevents data for another project or `All projects` from displaying under the wrong scope. |
| `projectScopeKey` uses a disjoint `all` or encoded `project:` representation. | A canonical project literally named `all` cannot collide with the unscoped cache identity. |
| A persistent Settings-filter change starts a new report-filter generation. | Changing visibility policy must invalidate cached and in-flight data from the old population. |
| Compare Periods filters A, B, history/coverage, and drill-downs with the same ID. | Report totals must derive from one consistent filtered corpus. |
| A Settings change that hides the active project clears the quick scope; an empty result does not. | The former protects visibility policy, while the latter is a valid period/provider result. |
