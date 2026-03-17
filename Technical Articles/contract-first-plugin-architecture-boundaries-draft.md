# Contract-First Plugin Architecture: Enforcing Boundaries Without Slowing Teams Down

How I structured activities, widgets, and harnesses with explicit contracts so the platform can grow without host-app rewrites.

## The Real Scaling Problem in Plugin Systems

Most plugin architectures do not fail because they lack extension points. They fail because the extension points are nominal while the real system still depends on hidden reach-ins, deep imports, and object-boundary exceptions.

That failure mode showed up quickly in this project. Once you have more than a few widgets and at least one activity that composes them, the dangerous shortcuts become obvious:

- harnesses start importing implementation files directly because it is faster in the moment,
- activities begin to depend on widget-specific internals instead of widget contracts,
- host apps reconstruct metadata from plugin folders instead of consuming exported registries,
- mode behavior drifts because permissions live in UI conditionals instead of typed runtime contracts.

The short-term result is speed. The medium-term result is boundary drift. And once boundary drift becomes normal, every new plugin increases integration cost for the whole platform.

That is why I moved the system toward a contract-first model. The goal was not abstract cleanliness. The goal was to make plugin growth predictable enough that new capabilities could be added without rewriting host code every time.

## Why This Architecture Mattered Here

The current implementation is intentionally narrow and concrete. One of the clearest examples is a headless equation-generation widget whose default configuration is explicitly math-first and standards-aware:

```ts
const DEFAULT_EQUATION_GENERATION_CONFIG = {
  subject: 'math',
  standards: ['8.EE.C.7'],
  description: 'Generate linear-equation prompts with worked-step expectations for grade 8.',
  generationPolicy: 'unique_per_assignment',
}
```

That tells you two things about the product reality.

First, the system is not pretending to be subject-agnostic from day one. The current implementation is grounded in a real use case: math workflows with explicit standards metadata. Second, the architecture cannot stay hard-coded to that use case if the platform is supposed to expand. If every new subject, generator, or evaluator requires host-specific rewiring, then "plugin architecture" is just branding over a monolith.

The design target was stricter than "make plugins possible." I wanted a system where a new widget or activity can be integrated through stable metadata and exported contracts, not through one-off host patches.

## Start With Object Boundaries, Not Components

The first step was naming the objects and making their boundaries explicit.

In this system, there are three separate plugin-related objects:

1. Activities orchestrate widgets.
2. Widgets own their own implementation details.
3. Harnesses simulate the host.

Those boundaries are not left to convention. They are written down in the project rules and repeated in the shared contracts.

The core rules are simple:

- activity harnesses are host-only,
- activities own widget orchestration,
- widgets own widget implementation details,
- non-widget code must access widgets only through exported APIs or metadata.

A separate harness contract turns that into an operational rule set. A harness must provide host inputs, capture host-observable outputs, and simulate student persistence. It must not implement widget orchestration logic or import raw implementation files.

That seems obvious until you look at how plugin systems usually decay. The harness is often where architecture shortcuts get normalized because it is "just dev tooling." I treated the harness as a boundary test instead. If the harness needs direct widget imports to make an activity work, the activity boundary is already wrong.

## The Widget Contract Is a Schema, Not a Vibe

The next step was making widget identity and taxonomy canonical.

The widget contract defines a shared `WidgetDefinition` shape with fields such as `widgetType`, `typeKey`, `typeId`, `functionKey`, `uiViews`, `resizeSync`, `inputSchema`, and `outputSchema`. A JSON schema then validates that structure.

That schema does a few useful things.

It fixes the runtime identifier around `widgetType`, so activities and hosts have one canonical lookup key. It locks `typeKey` and `typeId` together, so a widget cannot casually drift from `display-type` to `input-type` without an explicit contract change. It also constrains `functionKey` to a shared vocabulary such as `student-input`, `assessment`, and `authoring-aid`.

The important part is not the specific field names. The important part is that the taxonomy is centralized and reviewable.

In the activity types, that same contract becomes host-consumable metadata:

```ts
export interface AvailableWidgetDefinition {
  widgetType: string
  displayName: string
  typeKey: WidgetTypeKey
  typeId: 1 | 2 | 3 | 4
  role: WidgetRole
  submissionState?: WidgetSubmissionState
  iconUrl?: string
  uiViews?: WidgetUiViews
  resizeSync?: WidgetResizeSync
  generatedOutputFields?: GeneratedOutputFieldDefinition[]
  status?: 'ready' | 'stub'
}
```

That is the bridge between documentation and runtime. The written contract defines the authoritative taxonomy. The typed activity interface defines exactly what the host and activity are allowed to rely on.

## Typed Host Interfaces Make Integration Reviewable

An `exit-ticket` activity is where the contract-first approach becomes concrete.

Its host-facing type surface defines:

- mode as a closed union: `design | edit | preview | student | review`,
- a typed capability model,
- the design manifest shape,
- the implementation manifest shape,
- the student manifest shape,
- widget runtime and post-submit semantics.

That matters because the host relationship stops being implicit. The activity does not "just know" what the host will provide. It declares the dependency surface in code.

The props for `ExitTicketActivity` are explicit:

```ts
export interface ExitTicketActivityProps {
  mode: ActivityMode
  designManifest: ExitTicketDesignManifest
  implementationManifest: ExitTicketImplementationManifest
  assignmentId?: string
  assignedAt?: string
  studentManifest?: ExitTicketStudentManifest | null
  availableWidgets: AvailableWidgetDefinition[]
  widgetRegistry: Record<string, WidgetHostAdapter>
}
```

The runtime then enforces that contract instead of silently tolerating missing host state. The activity throws immediately if `availableWidgets` is empty or `widgetRegistry` is missing. In `preview`, `student`, and `review` modes it also throws if the host does not provide `assignmentId` and `assignedAt`.

That is the kind of friction I want. If a host integration is incomplete, I want the contract failure to be obvious at the boundary, not buried somewhere in rendering behavior.

## The Registry Is the Only Legal Access Path

The most important architectural move here is that widget consumption happens through exported APIs and registries, not through path reach-ins.

A public widget API re-exports the supported widget surface. That includes widget definitions like `scratchpadWidget`, public components like `ScratchPad`, and helper APIs such as `createEquationGenerationAssignmentPatches(...)`. The activity-side widget registry imports only from that public API and turns those exports into two host-facing objects:

- `EXIT_TICKET_AVAILABLE_WIDGETS`
- `EXIT_TICKET_WIDGET_REGISTRY`

`EXIT_TICKET_AVAILABLE_WIDGETS` is pure metadata. It tells the activity which widgets exist, how they are classified, what role they commonly fill, whether they are `ready` or `stub`, and which UI views they support.

`EXIT_TICKET_WIDGET_REGISTRY` is the behavior map. It binds a `widgetType` to the renderer, scoring, post-submit, and feedback callbacks the activity is allowed to use.

That split matters.

Metadata lets the activity compose and validate widgets without knowing their internals. The registry lets it invoke widget behavior through a stable adapter surface. Together, those two objects replace the fragile pattern of "just import the component you need from wherever it lives."

The equation-generation widget is a good example. It is classified as `data-generation-type`, marked with `generatedOutputFields`, and exposed through the registry with a `runOnAssignmentCreate` adapter that returns generic assignment patches. The activity consumes the result as contract data. It does not reach into the generator's private implementation to mutate other widgets directly.

## Harnesses Should Prove the Boundary, Not Bypass It

The exit-ticket harness follows the same rule.

At the integration layer, the harness imports only the public activity entry point and the widget registry:

```ts
import { ExitTicketActivity } from '@exit-ticket-activity'
import {
  EXIT_TICKET_AVAILABLE_WIDGETS,
  EXIT_TICKET_WIDGET_REGISTRY,
} from '@widget-registry'
```

That is the point of the harness contract. The harness acts like a host. It provides manifests, mode, student context, `availableWidgets`, and `widgetRegistry`. It captures output snapshots and student manifests. It does not orchestrate widgets itself.

The build configuration is just as important. The aliases resolve only to public entry points, not to private implementation modules. That sounds minor, but it removes a common loophole. If an alias resolves to internals, the boundary rule is meaningless.

This is also why I kept the scratchpad plugin's host contract narrow. Its distribution guide defines the host integration API as:

- `save()`
- `getPng()`
- `getSvg()`
- `clear()`

Everything else, including OCR and grading behavior, stays outside the plugin boundary. The host can evolve its analysis pipeline without turning the widget into an application-specific blob.

## Mode Gating Belongs in the Contract

A lot of plugin systems treat mode handling as UI logic. That is where behavior drift starts.

In the activity implementation, mode defaults are declared as a typed map from `ActivityMode` to `ActivityCapabilities`. That makes permissions reviewable. `design` can change structure. `edit` can change parameters but not structure. `preview` simulates student behavior without official submission. `student` can submit officially. `review` is read-only.

Because those capabilities are typed and centralized, the runtime can use them consistently instead of scattering permissions across JSX branches.

The submit flow uses the same approach. Widgets can declare `submissionState` values such as `required_pre_submit` or `post_submit`, and the activity executes post-submit adapters only for widgets that explicitly opt into that phase. Again, the point is not the specific field names. The point is that workflow semantics are part of the contract.

## Contract Evolution Is a First-Class Problem

Once contracts become the architecture, contract evolution becomes governance.

The widget contract already encodes that. The breaking-change policy explicitly treats changes to `widgetType`, `typeKey`, `typeId`, `functionKey`, required input fields, and required output fields as contract-breaking unless a backward-compatible path exists.

There is also a practical compatibility choice in the widget contract: `uiViews` is optional in the shared schema today so older widget definitions do not break immediately, but new and updated UI widgets are expected to include it. That is a small but important example of how to evolve a contract without surprising consumers.

I also keep the documentation update rule strict. If code-level interfaces change, the corresponding contract docs change in the same work session. Without that rule, the schema and the code drift apart, and the architecture eventually becomes aspirational documentation.

## What This Buys Me and What It Costs

The practical outcomes are straightforward.

Refactors get safer because the integration points are concentrated in documented contracts and registries. New widgets do not automatically imply host rewrites. Harnesses become useful because they validate host-facing behavior instead of secretly depending on implementation details. And the activity layer stays reusable across subjects because it consumes widget roles, metadata, and adapters rather than math-specific widget internals.

The tradeoffs are also real.

Contract-first systems demand more upfront type work, more review discipline, and more governance around what becomes public API. If you are careless, the public contract becomes the new dumping ground. The answer is not to relax the boundaries. The answer is to keep the public surface deliberately small and force new requirements through review.

## What I Would Add Next

The architecture is solid enough to grow, but the next improvements are obvious:

1. CI checks that validate widget definitions against the shared JSON schema.
2. Contract diff tooling for pull requests so breaking API changes are visible immediately.
3. Cross-plugin compatibility tests that exercise activities against registries and harnesses, not just isolated widgets.
4. A real third-party authoring guide that explains exactly how to ship a new widget or activity without depending on undocumented internal knowledge.

That last piece matters most if the platform is supposed to evolve beyond a math-first implementation. The internal contracts are already doing the heavy lifting. The next step is turning them into an external contribution model.

## The Main Lesson

The biggest mistake in plugin architecture is treating boundaries as a social preference instead of an executable design constraint.

The useful pattern was:

- define taxonomy and shared contract shape in docs and schema,
- mirror those contracts in typed host interfaces,
- export a public API,
- bind behavior through registries,
- force harnesses and hosts to consume only those public surfaces,
- fail fast when required host inputs are missing.

That approach is stricter than ad hoc plugin wiring, but it is also what makes the system extensible in practice. If outside contributors are ever going to add new content-generation, input, or evaluation plugins without patching the host application, this is the level of boundary discipline required.
