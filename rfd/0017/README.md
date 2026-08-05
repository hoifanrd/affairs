---
authors: hoifanrd
state: prediscussion
discussion:
labels: direction, security, ux
---

# [RFD] Capability Registry for User-Facing Permissions

This RFD introduces a **capability registry** in `core` — the centralized home of authorization per [RFD 0009](../0009/README.md) — as the single source of truth for user-facing permissions, and a convention by which the frontend renders UI affordances exclusively from server-computed capability booleans derived from that registry. A *capability* is a named action on an object type (`course.manage_roster`, `activities.grade`). Each capability is defined exactly once; both the route guard that enforces it and the permissions payload that advertises it are derived from that one definition, so drift between guard and payload is structurally impossible for registry-defined capabilities rather than discipline-dependent.

One residual drift surface — a UI site binding to the *wrong* capability — cannot be removed by construction; it is addressed under [Enforcement](#enforcement).

The motivating incident: the console rendered "Move to group" for instructors because the UI gated roster management on `capabilities.course.edit`, while the backend guards `POST /courses/:course_id/students` with the `admin` relation — a permission no instructor can reach. The affordance rendered, the action 403'd. Nothing in either repository could have caught this: the capability payload and the route guard are two hand-written artifacts with no structural link and no test tying them together.

Scope is the **authorization contract between `core` and the UI**: how capabilities are defined, enforced, advertised, consumed, and kept honest. The OpenFGA model itself (which relations exist, who holds them) is out of scope, as is the runtime enforcement path — every privileged action continues to go through the existing `authz` middleware and OpenFGA.

## Background

### How authorization reaches the UI today

`core` guards routes with an inline middleware call naming an OpenFGA relation and an object extractor:

```go
rg.POST(
    "/courses/:course_id/students",
    CreateCourseStudyingUser(courseMembershipService),
    a.Check(
        authz.RelationSystemAdmin,
        authz.WithParamContext(authz.ObjectTypeCourse, "course_id"),
    ),
)
```

The census of route authorization across the 8 API modules (`internal/api/*`, which run as separate extension processes per RFD 0009):

| Route authorization | Count | Notes |
|---|---|---|
| Route-level guard (`a.Check` / `a.Any`) | 212 | 27 distinct relation constants |
| No route-level guard | 60 | service-layer, in-handler, or proctoring session-bearer authorization |
| — of which in-handler FGA (proctoring) | 8 | one genuine cross-object OR (`GET /activities/:activity_id/config`: `activity#can_edit OR proctoring_activity#can_view_ops`); two more branch to a different *(relation, object)* pair by request shape |
| Lifecycle-gated (`a.Any` nested in a conjunctive outer `a.Check`) | 14 | ten OR the relation with two gates (exam window open, paper released for review), three with one gate, and the report formula route with three (adding score release) |
| Pure relation-OR (`a.Any`) | 1 | `GET /runs` (the pipeline-run list): `submission#can_read` on the delivery OR `collection#can_edit` on the collection |

The in-handler checks and the gate placements are invisible to any route-table reading, and a gated route's real grant is `outer relation AND (inner relation OR gates…)` — staff pass on the relation, students pass while the exam window is open or after the paper is released for review.

The UI learns what it may render from exactly one endpoint: `GET /courses/:course_id/permissions` (`internal/api/logistics/course_service.go`), which fires six FGA checks and hand-maps them onto seven booleans (`course.{edit,view_members}`, `activities.{create,grade}`, `documents.create`, `collections.{create,submit}`) plus a display `role`. The mapping is by hand; some booleans deliberately alias other checks to save round trips (`activities.grade → can_edit`).

On the frontend (`apps/console`, in the `ui` repo alongside the student-facing `apps/client`), an inventory found roughly **230 permission-gated lines across 43 source files** (sweep for permission-shaped identifiers — `capabilities`, `isStaff`/`isStudent`, `canManage`, `is_admin`, `PermissionDenied` — story and test files excluded), of which only **two** capabilities are actually consumed: `course.edit` and `activities.create`.

Everything else rides on `isStaff := capabilities.course.edit`, a lossy alias that stands in for grading, execution configuration, proctoring configuration, score release, recordings review, and roster management. That derivation is triplicated (one context provider plus two SSR loaders, linked only by a "mirrors the derivation in…" comment), and below the three provider mount points, gating is ad-hoc boolean props threaded through components (`canManage` alone threads into ~13 conditional render sites across two components).

There is no `<Can>`-style abstraction, no capability constants, and no story or test exercising a permission-denied state *driven by the capability payload* — the two read-only roster stories flip a `canManage` prop directly, which is precisely the layer below where this bug lived.

Role and permission management is also a standing frontend functional guarantee (RFD 0005, frontend QAS, still on branch); this RFD specifies the contract that guarantee runs on, not the roles themselves.

### Why it drifted

1. **Two hand-written artifacts, no link.** The capabilities mapping and the route guards are maintained independently. The roster routes are guarded by `admin`; the payload has no boolean that reflects `admin`; the UI comment above its gate asserted (wrongly) that instructors may manage the roster. Each artifact was locally plausible; the contradiction lived between them.
2. **Coverage gap.** One capabilities endpoint (course-scoped, 7 booleans) versus 212 guards — 182 of them outside the logistics module: 169 on `activity`/`examination`/`collection`/`submission`/`proctoring_*`/`pipeline`/`formula`/`environment` objects with no capability surface at all, the remaining 13 checking `course` or `system` from other modules. Where no capability exists, UI authors reach for whatever boolean is nearby (`isStaff`), and every such substitution is a latent instance of this bug.
3. **No enforcement.** `core`'s route-wiring tests count guard invocations per relation (`.Times(N)` with `gomock.Any()` extractors) but cannot see the route-to-relation pairing, and `internal/authz/README.md` itself documents their fragility. `types.gen.go` (generated from `model.openfga`) is not covered by the CI drift lint (`scripts/generate_lint.sh`), so even model-to-constants drift would pass CI today. The UI has no lint, test, or story that would notice a gate reading the wrong signal.
4. **Naming traps.** Relation strings are not unique keys: `"admin"` names five relations across types and `"can_edit"` nine. `RelationSystemAdmin` and `RelationCourseAdmin` are the same string, which is how a course-object guard came to be written with a system-named constant. Any convention keyed on relation strings alone will collide; the key must be *(object type, action)*.

### Prior art

This problem is well-trodden, and the industry answer is consistent:

- **[GitLab](https://docs.gitlab.com/development/api_graphql_styleguide/)** exposes `userPermissions` fields on GraphQL types via `expose_permissions`, where the advertised booleans are abilities "defined in our policies" — the same `DeclarativePolicy` system that [authorizes mutations](https://docs.gitlab.com/development/graphql_guide/authorization/). Drift is prevented structurally, not by testing — the boolean the UI reads and the guard the mutation runs are the same policy definition.
- **[GitHub](https://docs.github.com/en/graphql/reference/objects#repository)** ships per-action viewer booleans (`viewerCanUpdate`, `viewerCanAdminister`) on GraphQL objects. The coarse `viewerPermission` role enum is nullable — documented to return null when authenticated as a GitHub App — and cannot express per-action rights, so the fine-grained `viewerCan*` booleans are the dependable contract UI code branches on.
- **[OpenFGA](https://openfga.dev/docs/modeling/roles-and-permissions)**'s modeling guidance is to check *fine permissions, not roles* — so role restructuring in the model requires no application changes — and its [relationship-queries guidance](https://openfga.dev/docs/interacting/relationship-queries) for UI show/hide is a per-object batch of checks (`BatchCheck`), not per-widget calls.
- OpenFGA's SDKs also offer `ListRelations` ("list the relations a user has on an object"), the SDK-native per-object capability vector — implemented over client-side `ClientBatchCheck` (parallelized single checks), not the server-side batch API. The registry proposed here supersedes it for our case rather than wrapping it: `ListRelations` returns raw relation names (the client would need the relation vocabulary, which rule 1 keeps server-side) and cannot express lifecycle gates or OR-grants.
- **[Oso](https://www.osohq.com/docs/learn/guides/ui)** and **[Cerbos](https://docs.cerbos.dev/cerbos/latest/recipes/ui.html)** (authorization vendors) both publish the same recipe for frontends: the backend evaluates permissions and returns them alongside the resource; the frontend renders from them; the backend check on the actual action remains the enforcement. Cerbos warns explicitly that exposing a raw check API to browsers lets an attacker "probe your authorization policies"; Oso's guidance likewise forbids calling the authorization service from the frontend (untrusted client code, credential exposure).
- **[OWASP A01](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)** is categorical that access control is only effective in trusted server-side code.
- **[Zanzibar](https://www.usenix.org/system/files/atc19-pang.pdf)** (the paper OpenFGA implements) serves the vast majority of checks from locally replicated, staleness-bounded data — staleness-tolerant ("Safe") checks outnumber freshness-requiring ("Recent") ones by roughly two orders of magnitude (§4.1). Permission bits rendered in a UI are allowed to be seconds-stale by design; the authoritative check happens on the mutation.

In-repo, RFD 0009 already fixes authentication and authorization as core-owned rather than per-extension ("Security infrastructure must be centralized and consistent"), and [RFD 0011](../0011/README.md) is today's one case of an extension enforcing OpenFGA in its own handler middleware — the two shapes of guard this registry has to span. The house precedent for a cross-repo vocabulary contract is the semantic fault taxonomy spec in `core` (`docs/superpowers/specs/2026-07-15-semantic-fault-taxonomy-design.md`): a single vocabulary defined in `core`, carried to the UI through the existing `swag → docs/swagger.json → sdk.yml dispatch → kubb regen` pipeline, with an explicit deploy-ordering rule.

## Scope

### In scope

- **The capability registry** in `core`: shape, naming rules, resolution semantics.
- **Guard convention**: route guards reference registry entries rather than raw relations.
- **Permissions endpoints**: per-scope payloads generated from the registry, evaluated via BatchCheck.
- **The SDK contract**: how capability names and payload types flow to the UI (existing swagger pipeline).
- **UI consumption convention**: a single permission layer, lint enforcement, storybook/test conventions, 403 handling.
- **CI enforcement** in both repos.

### Out of scope

- **The OpenFGA model.** No relation is added, removed, or re-derived by this RFD. (Whether e.g. roster management should be `admin`-only or `can_edit` is a product/policy decision that this RFD makes *visible*, not one it takes.)
- **`apps/client` (student exam client).** It deliberately has no permission model: it reacts to 403s, and treats them as "not yet available" on release axes (score release and paper release are independent, and per-course flags cannot express the mixed TA-and-student case). This is documented behavior and stays.
- **The invigilation cockpit role.** `CockpitRole` is URL-derived by explicit requirement (the cockpit must function without a roster fetch); the backend enforces via room-scoped checks in the proctoring extension (RFD 0011 and its follow-up proctoring extension RFD). It remains a documented, deliberate second system.
- **Machine-to-machine authorization** (`transport.Internal` bypass) and the proctoring session-bearer path.

## Terminology

Terms used throughout this RFD:

- **Capability** — a named action on an object type (`course.manage_roster`), the unit of this contract.
- **Scope** — the resource whose permissions endpoint serves a set of capabilities (`course`, `activity`).
- **Path** — a capability's field position in its scope's wire payload (`activities.create` inside the course payload).
- **Lifecycle gate** — a non-relation condition ORed into a grant (exam window open, paper released for review, score released), owned by the module whose routes it protects.
- **OR-grant** — a grant satisfiable by more than one *(relation, object)* pair.
- **Affordance** — a UI element offering an action: a button, a menu item, an editable field.
- **Permission layer** — the single UI module through which all capability reads flow.
- **Expand/contract** — additive-first contract evolution: add the new field, migrate consumers, deprecate, then remove.

## Proposal

The convention is six rules:

1. **A capability is a named action on an object type**, defined exactly once in `core`'s capability registry as *(scope, payload path, relation-on-object[, gate])*. The payload path's leaf key is verb-first (`manage_roster`, `grade`, `submit` — as in `course.manage_roster`, `activities.grade`); paths are unique within a scope. The underlying relation string is an implementation detail that never leaves `core`.
2. **Route guards reference capabilities, not relations.** New and migrated routes use `capability.Require(a, capability.CourseManageRoster, "course_id")`; the guard resolves both the relation and the object type from the registry. Raw `a.Check(authz.RelationX, …)` is legacy, to be removed module-by-module.
3. **Permissions payloads are generated from the registry.** A scope's `/permissions` endpoint iterates the registry's capabilities for that scope and evaluates them with the same resolution the guard uses. Adding a registry capability extends the guard vocabulary and the payload in the same commit; for registry-defined capabilities it is impossible to add one without the other. (Routes not yet migrated off raw `a.Check` are outside this guarantee — see [Enforcement](#enforcement).)
4. **The frontend renders only from capability booleans.** Never from `role` (display-only, matching `core`'s own rule for the JWT layout claim: "Embedded role is display-only; every privileged action must still go through OpenFGA"), never from client-side policy reasoning, never from a nearby boolean that happens to correlate. Feature flags are a separate axis: never encode feature availability as a capability, and never wire flags through the permission layer.
5. **Capabilities are a UX cache; the 403 is the enforcement.** Staleness is acceptable and expected. Client-side, a 403 on an action the cache said was allowed triggers a refetch of that scope's permissions, not an error report; server-side, 403s continue to be logged (and alerted on repetition) per OWASP guidance — the [RFD 0008](../0008/README.md) telemetry stack is where that logging and alerting lands. The rule governs UX, not auditing.
6. **All UI capability reads go through one permission layer** — including SSR loaders. Raw `capabilities?.x?.y` property access outside the layer is lint-banned, as is `role ===` comparison outside the display badge.

### Why this shape

Each load-bearing decision, and what justifies it:

- **Single definition in `core`, both consumers derived (rules 1–3).** The motivating bug lived *between* two hand-written artifacts; no test caught it because each was locally consistent. GitLab's `expose_permissions` demonstrates the only robust fix: the advertised boolean and the enforcing guard call the same definition, so consistency is a property of the construction, not of review discipline. We considered and rejected a consistency *test* between two independent artifacts — it must itself be maintained against both sides, and the existing `.Times(N)` guard tests have already rotted into exactly that failure mode.
- **Server-computed booleans, not client-side evaluation (rule 4).** The policy inputs are OpenFGA tuples (other users' enrollments and staff assignments) that must not ship to browsers; OWASP A01 and every vendor recipe in this space (Oso, Cerbos, Permit.io) converge on server-computed decisions with the client as a cache. This is also why the payload, not a raw check proxy, is the interface: a proxy would relocate the route-to-relation mapping into TypeScript and open a policy-probing surface.
- **Per-scope capability vectors, not per-endpoint permissions.** The UI gates a couple hundred sites on affordances ("may I manage this roster"), not on routes; routes outnumber affordances by better than an order of magnitude — 212 guards against 7 payload booleans today, perhaps 15 once the `isStaff` alias is split — and churn faster. GitHub's contract is instructive: the coarse role enum proved unable to express per-action rights (and is nullable for token-scoped clients); the per-action booleans are what UI code branches on. Batch evaluation per scope follows OpenFGA's own UI guidance (BatchCheck for show/hide vectors).
- **Fine capabilities over roles (rules 1, 4).** OpenFGA's modeling guidance: applications should check permissions, not roles, so the model can be restructured without application changes. The incident is the counterexample to role reasoning — "instructors manage rosters" was role-plausible and grant-false.
- **Staleness tolerated, 403 as backstop (rule 5).** Zanzibar itself serves the vast majority of checks from staleness-bounded replicas; the authoritative decision happens on the mutation. Fighting staleness in the UI cache buys nothing the backstop doesn't already guarantee.
- **A lint-enforced single read path (rule 6).** The audit found the convention-by-culture alternative already failed here: three duplicated derivations, a lossy `isStaff` alias across the console, and zero capability-seeded stories or tests. A convention that tooling does not enforce regresses to whichever boolean is nearest. Enforcing authorization conventions with custom lint rules is precedented at scale (GitLab ships custom RuboCop cops for exactly this, e.g. [`Graphql/AuthorizeTypes`](https://gitlab.com/gitlab-org/gitlab/blob/master/rubocop/cop/graphql/authorize_types.rb)).

## The capability registry

One definition fans out to both consumers, and the 403 remains the backstop:

```
model.openfga ──authzgen──► relation constants
                                   │
         capability registry: (scope, path, relation on object[, gate])
                  │                                     │
                  ▼                                     ▼
    Require(a, c, "course_id")            GET /<scope>/:id/permissions
    guard on the mutating route           BatchCheck over the scope's entries
                  │                                     │
                  ▼                                     ▼
            OpenFGA Check                 swagger.json ──kubb──► @zinc/sdk
                  │                                     │
                  ▼                                     ▼
    403 = enforcement backstop ◄───────── useCapability("course.manage_roster")
                                          renders or hides the affordance
```

A new package `internal/authz/capability`:

```go
package capability

type Capability struct {
    Scope    string   // which permissions endpoint serves it: "course", "activity"
    Path     string   // JSON path within that payload: "course.edit", "activities.create"
    Relation string   // OpenFGA relation, e.g. authz.RelationCourseAdmin
    Object   string   // authz.ObjectType the relation is checked against
    Gate     GateFunc // optional lifecycle gate, registered by the owning module; nil for pure-FGA
}

var (
    CourseEdit         = define("course", "course.edit",          authz.RelationCourseCanEdit,           authz.ObjectTypeCourse)
    CourseViewMembers  = define("course", "course.view_members",  authz.RelationCourseCanListStudents,   authz.ObjectTypeCourse)
    CourseManageRoster = define("course", "course.manage_roster", authz.RelationCourseAdmin,             authz.ObjectTypeCourse)
    ActivityCreate     = define("course", "activities.create",    authz.RelationCourseCanCreateActivity, authz.ObjectTypeCourse)
    // ...
)
```

Notes:

- `define` registers into a package-level table keyed by *(scope, path)* and panics on duplicates at init, so the registry cannot silently fork. It returns `*Capability`, so a later `RegisterGate` mutates the registry entry rather than a copy. `Path` is the position in the wire payload, which is how today's nested shape (`course.*`, `activities.*`, `documents.*`, `collections.*` under the course endpoint) is reproduced exactly — existing field names are preserved verbatim, so the current UI keeps working through every phase of the migration.
- The registry names the relation *honestly with respect to the object it is checked against*: the roster capability is defined with `RelationCourseAdmin` on `ObjectTypeCourse` — the same string the current guards check, but no longer written as `RelationSystemAdmin` against a course object (a string-collision accident the current code contains).
- **Aliased capabilities are dissolved.** Today `activities.grade`, `documents.create`, and `collections.create` silently alias `can_edit` to save FGA calls. Under the registry each gets its own honest definition; the round-trip cost concern is answered by BatchCheck (below), not by aliasing. The fourth alias is the most role-shaped: `collections.submit` aliases the `student` **role** relation, while its honest definition is `collection#can_submit` on a *collection* object — not course-scoped at all. That one field is the first concrete instance of the cross-scope question in [Discussion](#discussion).
- OR-grants — today one proctoring in-handler check across two object types, plus one route-level relation-OR (`GET /runs`, the pipeline-run list: `submission#can_read` on the delivery OR `collection#can_edit` on the collection) — are representable as `AnyOf(capA, capB)` entries whose resolution requires resolving related object IDs. This is deliberately deferred (see [Rollout](#rollout)); the registry shape accommodates it, but no proctoring capability is defined in the initial phases.

## Guard middleware

`Require` is thin sugar over the existing `a.Check`. It lives in the `capability` package, keeping the dependency one-directional (`capability → authz`) and leaving the three-method `authz.Middleware` interface — its generated mock and its eight hand-written test doubles — untouched:

```go
// package capability
func Require(a authz.Middleware, c *Capability, param string) echo.MiddlewareFunc {
    check := a.Check(c.Relation, authz.WithParamContext(c.Object, param))
    if c.Gate == nil {
        return check
    }
    return a.Any(check, c.Gate()) // the same disjunction the gated routes use today
}
// variants mirror the existing body-/query-sourced extractors
```

Because `Require` builds the extractor from the registry's object type, a call site cannot pair a capability with the wrong object — the string-collision trap that produced `RelationSystemAdmin`-on-course is closed by construction, not convention. Semantics are unchanged: one FGA check per guard per request, dev-mode behavior, `Any` composition, typed-fault passthrough, and the `transport.Internal` bypass all stay as they are. Migration is mechanical and can proceed route-file by route-file; both forms coexist indefinitely.

Lifecycle gates are the one place a guard is *more* than one relation, and the gated routes deserve precision:

- **The full grant is a conjunction around a disjunction.** All fourteen gated routes have the shape `outer a.Check AND a.Any(inner relation, gates…)` — the exam-document routes require `can_read_questions` and then `can_edit OR window-open OR paper-released-for-review`. Ten of the fourteen OR the relation with those two gates, three with one, and the report formula route with three (adding score release). A capability modeling only the disjunct is *not* the whole grant.
- **Gates are module-owned middleware.** They are `echo` middleware reading route params, and their owners are separate extension processes per RFD 0009 — `report` deliberately cannot import `examination`'s gates, and one gate makes a NATS round-trip with a 3-second timeout.
- **The registry never imports gates.** The owning module registers its gate against the capability at init — `capability.RegisterGate(ExamDocumentRead, m.CheckDocumentCollectionTime)` (a phase-4 activity-scope entry, not among the course-scope examples above) — and `Require` composes it as shown.
- **The outer conjunct and the payload cost are phase-4 problems.** Representing the conjunct, and paying gate evaluation on the payload path where the gate's route-param inputs don't exist, are deferred to the cross-scope and deny-reason questions in [Discussion](#discussion) — which is why the rollout treats phase 4 as evaluative rather than committed.

## Permissions endpoints

One endpoint per scope, uniform shape, generated from the registry:

- `GET /courses/:course_id/permissions` — **exists**; reimplemented to iterate registry entries with `Scope == "course"` (response shape and field names unchanged, plus new fields such as `course.manage_roster`).
- `GET /activities/:activity_id/permissions` — **new**; covers the activity/examination cluster where the UI's `isStaff` alias currently does the most damage (grading, execution, score release, proctoring configuration are distinct backend relations today, indistinguishable to the UI).

Implementation notes:

- **BatchCheck.** The pinned OpenFGA go-sdk (v0.8.0) exposes server-side `BatchCheck` (added in sdk v0.7.0; **requires OpenFGA server ≥ 1.8.0**) alongside `ClientBatchCheck`, which merely parallelizes single checks client-side — no better than today's errgroup. `core` uses neither yet. A scope's N capabilities become one server-side BatchCheck request where the deployment allows, with the errgroup pattern as fallback; lifecycle gates evaluate app-side after the batch returns.
- **Existing semantics preserved**: authn-only (no FGA guard on the endpoint itself), non-members receive all-false rather than 403 (an authenticated principal with no tuples simply gets every boolean false), resource existence check first (404 for missing resource), `Cache-Control: no-store, private`. Sessionless callers — including the internal machine transport, which bypasses `a.Check` but not `authn.RequireUserContext` — are 401'd today; whether they should instead receive all-false is an open point in [Discussion](#discussion).
- **Gated capabilities** (phase 4) return the disjunction the guard's `a.Any` evaluates (`relation OR gates…`), ANDed with the route's outer relation (previous section). Three caveats:
  - `a.Any` is three-valued: an *undecided* branch (a `service.*` fault from a gate's dependency) outranks a denial and yields 503, not 403. The payload can only ship the two-valued projection, and rule 5's 403-refetch does not cover that 5xx case.
  - The guard records *which* branch granted (the examinee- and review-gate context flags that reshape exam layout today); a boolean erases that, and also erases *why* it is false. Surfaces rendering a gated capability's false state therefore use "not available" copy, never "no permission" ([Discussion](#discussion) weighs a per-capability reason field).
  - Lifecycle gating on the data side is already specified — [RFD 0013](../0013/README.md)'s stage `visibility` filters hide artifacts `until = "collection_stop"` and fail closed when no window exists. A gated capability's `false` is the affordance-side projection of the same window.
  - Until a scope's gates are migrated, gate-dependent capabilities are simply not defined in the registry, rather than defined dishonestly.
- `role` stays in the course payload as display-only data.
- **Deliberately not embedding** a `viewer` block inside resource GET responses (GitHub-style) for now: separate endpoints match the existing pattern, cache independently under react-query, and keep resource handlers free of authz fan-out. Revisit if SSR waterfalls demand it (see [Discussion](#discussion)).

## The SDK contract

No new machinery. Capability names *are* the JSON field names of the permissions response types; the existing pipeline (`swag` → `docs/swagger.json` → `core`'s `sdk.yml` dispatches the UI regen workflow → kubb generates types/zod/hooks) already delivers typed keys to the frontend. The generated Go response structs per scope are emitted from the registry (a small generator extension in the spirit of the existing `authzgen`), so the swagger schema is itself registry-derived — removing one more hand-maintained artifact (the course permissions handler's godoc already fails to document the 404 its service returns; generated annotations don't rot).

**Contract evolution is expand/contract.** Adding a capability is always safe (additive field). Removing one is two-step: mark the field `deprecated` in the spec first (as [GitLab](https://docs.gitlab.com/development/api_graphql_styleguide/) and [GitHub](https://docs.github.com/en/graphql/overview/breaking-changes) deprecate GraphQL fields before removal), remove only after the UI consumer is gone.

Optionally, later: `@x-zinc-capability` operation annotations in handler godoc so the API docs state which capability each operation requires (swag v2 passes arbitrary `@x-*` JSON through; note the emitted spec is Swagger 2.0 — the OAS3 conversion happens in the UI's regen workflow). This is documentation only — **explicitly not the sync mechanism**, because annotations live on handler godoc while guards live in route registration, which would recreate the drift surface this RFD exists to remove.

## UI consumption convention

A single permission layer in `apps/console`:

- **`PermissionsProvider(scope, id)`** — owns fetching (existing generated hooks), caching (`staleTime` ~5 min as today), the loading tri-state (every gated route currently re-implements `isStudent && !isLoading` by hand), and 403-triggered invalidation.
- **`useCapability("course.manage_roster")`** — the only sanctioned read path; typed keys from the generated SDK types.
- **`<RequireCapability cap fallback>`** — thin wrapper for the common render-or-fallback case (the four staff-only routes that hand-roll `!isStaff && !isLoading → <PermissionDenied/>` collapse into it).
- **SSR loaders go through the same layer**: a loader-usable helper shares the provider's query keys and derivations — killing the current triplication rather than relocating it — and the lint ban covers loader code.
- **`isStaff` survives as a framing concept only** (which shell to render), derived in exactly one place; every *action* affordance binds to its own capability. The `canManage`-style prop threading below the layer is fine and stays — the rule governs where the value comes from, not how it travels.
- **Lists are filtered server-side.** Capabilities answer "may the viewer act on *this* object"; *which objects appear* is server-side query scoping — `GET /courses` filters by an FGA `ListObjects` query, `/me/coursework` by a SQL membership CTE; either mechanism is fine, the point is that it happens server-side. Capability booleans never filter list rows client-side.
- **Lint enforcement**: a CI check (a real biome/ESLint rule rather than a grep gate, so loaders and tests are covered) banning `capabilities?.` member access outside the permission layer and `role ===` comparisons outside the badge.
- **Storybook/tests**: the story harness learns to seed permissions query data (the `seed(queryClient)` seam already exists); every permission-gated surface gets a denied-state story/variant driven by the capability payload. Today zero stories mock capabilities — the two read-only roster stories flip a `canManage` prop, one layer below where the motivating bug lived, so it was invisible to every UI test.
- **403 UX**: mutation error mapping classifies `AccessInsufficientPermission` distinctly (today a 403 in `runBulkMutation` is indistinguishable from a network failure), and a 403 from a capability-gated action invalidates that scope's permissions query (rule 5). Because FGA reads are staleness-tolerant, the refetch may still report the capability as granted — surface the failure once and re-render from the refetched payload; never auto-retry the mutation.

## Enforcement

- **Guard-payload consistency needs no test for migrated routes** — both derive from the registry; that is the point of the design. Until a module migrates, the coverage test below is what pins its guards; a capability whose scope still has routes on raw `a.Check` is reported as *unmigrated* by that test, not treated as proven.
- **The residual drift surface is site-to-capability binding.** The registry cannot force a UI site to read the *right* capability — a site could gate roster on `course.edit` while the route requires `course.manage_roster`, and GitLab's design has the identical residual gap. Mitigations: denied-state stories per gated surface (above), and — worth evaluating in phase 3 — a registry-derived route-to-capability map emitted into the SDK (derived from route registration, **not** godoc annotations), so generated mutation hooks know their required capability and the permission layer can warn in development when a rendered affordance and the mutation behind it disagree.
- **Registry coverage test** (`core`): walk `e.Routes()` and assert every mutating route is guarded by `Require`, by legacy `a.Check` (tracked, and reported as unmigrated), or carries an explicit service-layer-authz registration marker. This replaces the brittle relation-count `.Times(N)` pattern (whose fragility `internal/authz/README.md` already documents) by generalizing the recording-middleware approach two tests already use ad hoc (`examination/import_manifest_route_test.go`, `report/route_collision_test.go` — the latter pins *(relation, object)* per route).
- **Close an adjacent CI gap** (found during the audit): `scripts/generate_lint.sh` does not regenerate/diff `types.gen.go`, so a `model.openfga` edit without regen passes CI today. Add it alongside the new registry-derived artifacts.
- **Cross-repo rollout rule**, per the semantic-fault precedent: `core` deploys first only when the corresponding UI release lands in the same window; capability removals follow the expand/contract sequence above.

## Rollout

Each phase is independently shippable; PRs target `develop` in their respective repos.

| Phase | Content | Repo(s) |
|---|---|---|
| 0 | Interim fix for the motivating bug: console gates roster mutations on the existing `user.is_admin` from `GET /users/me` — today equivalent to the guard's `course#admin` only because `course.parent` is `[system]` (`model.openfga` notes "using system instead of department for now") — pending phases 1–2 | `ui` |
| 1 | `capability` package incl. the `Require` guard helper; migrate the logistics module's guards; reimplement the course permissions endpoint from the registry (server-side BatchCheck where the OpenFGA deployment is ≥ 1.8.0); add `course.manage_roster`; `generate_lint.sh` covers `types.gen.go` and registry-derived artifacts | `core` |
| 2 | SDK regen; UI permission layer (`PermissionsProvider` / `useCapability` / `<RequireCapability>` / loader helper); console course-scope gating migrates to it (roster gate moves from `is_admin` to `course.manage_roster`); lint rule; storybook seeding + denied-state stories; 403 classification in mutations | `ui` |
| 3 | `GET /activities/:activity_id/permissions`; migrate examination/submission/report guards; console splits the `isStaff` alias into real capabilities (grade, release, configure, review); evaluate the SDK route-to-capability map ([Enforcement](#enforcement)) | `core` → `ui` pair |
| 4 | Lifecycle-gate-aware capabilities (exam window, paper review, score release) with the "not available" copy rule — gated surfaces that cannot be honestly modelled stay 403-backstopped; registry coverage test replaces `.Times(N)` wiring tests module-by-module; evaluate proctoring scope (`AnyOf` support, over the object types RFD 0011 introduced) — may reasonably remain 403-backstopped indefinitely | `core` |

Phases 1–2 eliminate the drift class that caused the motivating bug for everything course-scoped. Phase 3 removes the `isStaff` lie, which the audit identified as the largest latent-bug surface: any future backend split of staff permissions is currently invisible to the UI.

## Abandoned ideas

Alternatives we considered and set aside, recorded so the same paths aren't re-walked.

### A generic batch-check proxy endpoint for the UI

"Let the UI ask `can I <relation> on <object>?` directly." Rejected: it moves the route-to-relation mapping *into the client* — the drift problem reappears one layer down, now maintained in TypeScript — and it exposes the policy-probing surface Cerbos warns against explicitly (Oso's frontend guidance forbids direct authorization-service calls for related reasons). It also cannot express lifecycle gates or the in-handler OR grant.

### Client-side policy evaluation (CASL-isomorphic rules, embedded WASM PDP)

Rejected: the policy lives in OpenFGA relationship tuples, so client-side evaluation would mean shipping tuple data (other users' enrollments and staff assignments) to the browser. OWASP A01 is categorical that access control logic is only effective server-side; every vendor in this space that ships a frontend SDK (including Permit.io's CASL integration) has the server compute decisions and the client merely cache them — which is precisely what capability payloads are.

### OpenAPI `x-required-permission` extensions as the sync backbone

Encoding the required permission per operation in the swagger spec and generating both the Go guard wiring and TS constants from it. Attractive on paper (oapi-codegen has precedent for scope-carrying codegen), but rejected: swag reads annotations from handler godoc while guards live in `Register*Routes` — nothing forces them to agree, so the spec becomes a third artifact that can drift; there is no off-the-shelf tooling for the TS side; and the UI does not actually need per-*endpoint* permissions — it needs per-*resource* capability vectors, which are fewer and stabler than routes. Kept only as optional documentation sugar.

### Hand-extending the current capabilities endpoint as needed

The status quo plus discipline. Rejected: this is exactly the process that produced the motivating bug, and it leaves the ~180 non-logistics guards essentially without a capability surface, guaranteeing more `isStaff`-style substitutions.

### Role-based UI gating

Deriving UI affordances from `role` ("instructor sees X"). Rejected on OpenFGA's own modeling guidance (check permissions, not roles, so the model can be restructured without application changes) — and by this incident, where role-shaped reasoning ("instructors manage rosters") contradicted the actual grant.

## Known limitations and accepted trade-offs

- **Site-to-capability binding remains a human choice.** The registry removes drift between guard and payload; it cannot force a UI site to bind the right capability. Accepted as residual, with the mitigations under [Enforcement](#enforcement).
- **A gated boolean is a two-valued projection of a three-valued guard.** An undecided gate branch yields 503 at the route while the payload can only say false, and the granting-branch identity (examinee vs review gate) is not carried. Accepted with the "not available" copy rule; a reason enum can be added additively later (see [Discussion](#discussion)).
- **Capability payloads are staleness-tolerant by design.** A ≤ 5-minute-stale boolean may render an affordance the mutation then 403s; the backstop plus refetch-on-403 is the accepted answer — the same shape of trade-off RFD 0011 accepts for mid-session permission revocation within its token TTL window.
- **Roster policy is made visible, not decided.** The registry makes the current grant explicit (`manage_roster = admin`); whether that is the *intended* policy (versus `can_edit`, i.e. coordinators/instructors) is a product decision this RFD does not take. If it changes later, it changes in exactly one place.
- **`GET /permit` and the JWT `layout` claim stay exempt.** Both are adjacent role-shaped signals; the latter is explicitly display-only. They are documented as exempt display machinery rather than re-expressed through the registry.

## Discussion

1. **Capability granularity.** One `course.manage_roster` for all six roster routes (enroll/remove × student/staff, group create/delete), or split? Proposed: one, until a product need splits it — capabilities track *affordances*, not endpoints, and an affordance-per-endpoint registry would just be the route table again.
2. **Embedded viewer blocks.** Should resource GET responses eventually embed their capability vector (GitHub-style) to save a round trip on first paint, with the separate endpoints retained for refetch? Proposed: revisit after phase 3 with SSR timing data.
3. **Cross-scope capabilities.** Capabilities whose relation lives on a different object type than their scope: `course`-scope `collections.submit` is honestly `collection#can_submit` on a collection, and `activity`-scope capabilities lean on `examination`/`collection`/`proctoring_activity` objects (parent-linked in the model). Checked against which object ID, resolved how, and does the payload expose the seam? A phase-3/4 design detail; flagged here because it is where the "one scope = one object type" simplification will be tested.
4. **Deny reasons for gated capabilities.** A gated capability's `false` conflates "no permission" with "window closed". Is the "not available" copy rule sufficient, or should gated entries carry a reason enum? GitHub's `viewerCannotUpdateReasons` (ARCHIVED / LOCKED / INSUFFICIENT_ACCESS, …) is precedent both that mature products eventually need reasons and that the enum can be added additively later. Proposed: copy rule now; a reason field only if a surface demonstrably needs the distinction (`apps/client`, which needs it today, is out of scope and already handles it via 403 semantics).
5. **Sessionless and machine principals.** The permissions endpoints 401 sessionless callers today (`authn.RequireUserContext`; the internal transport bypasses `a.Check` but not authn). Whether they should instead receive all-false — degrading without an error, as GitHub's `viewerPermission` returns null for app-scoped tokens — is open; nothing in the phases depends on it.

## References

- [RFD 0008 — Implementing Telemetry in the ZINC Grading System](../0008/README.md)
- [RFD 0009 — ZINC Extension System](../0009/README.md)
- [RFD 0011 — Real-Time Video Invigilation Using LiveKit](../0011/README.md)
- [RFD 0013 — Pipeline v3 and Formula Redesign](../0013/README.md)
- GitLab GraphQL: [`expose_permissions` styleguide](https://docs.gitlab.com/development/api_graphql_styleguide/), [GraphQL authorization](https://docs.gitlab.com/development/graphql_guide/authorization/), [RuboCop `Graphql/AuthorizeTypes`](https://gitlab.com/gitlab-org/gitlab/blob/master/rubocop/cop/graphql/authorize_types.rb)
- GitHub GraphQL: [Repository object (viewer fields)](https://docs.github.com/en/graphql/reference/objects#repository), [breaking-changes process](https://docs.github.com/en/graphql/overview/breaking-changes)
- OpenFGA: [relationship queries (BatchCheck for UI)](https://openfga.dev/docs/interacting/relationship-queries), [roles and permissions](https://openfga.dev/docs/modeling/roles-and-permissions), [go-sdk](https://github.com/openfga/go-sdk) (server-side BatchCheck since v0.7.0, requires OpenFGA ≥ 1.8.0; `ListRelations` quote and `ClientBatchCheck` delegation from its README and `client/client.go`)
- Oso: [Best Practices for Frontend Authorization](https://www.osohq.com/docs/learn/guides/ui)
- Cerbos: [Using Cerbos in the UI](https://docs.cerbos.dev/cerbos/latest/recipes/ui.html)
- Zanzibar: [Google's Consistent, Global Authorization System, USENIX ATC '19, §4.1](https://www.usenix.org/system/files/atc19-pang.pdf)
- OWASP: [A01:2021 Broken Access Control](https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/)
- Semantic fault taxonomy spec in `core` (`docs/superpowers/specs/2026-07-15-semantic-fault-taxonomy-design.md`) — house precedent for a core-to-UI vocabulary contract
