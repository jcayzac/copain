# Rust Component System for an SSG Framework

## Purpose

This document describes a proposed Rust component-system library stack for use by an outer SSG framework. It is both an implementation plan and an early integration guide.

Companion document:

- See [component-system-api-proposal.md](/Users/julien.cayzac/github/jcayzac/copain/docs/component-system-api-proposal.md) for a provisional API sketch. That document is explicitly a proposal, not a definitive plan or stable API commitment.

The component-system crates should provide:

- component semantics
- HTML rendering
- metadata and manifests
- declaration/type models for client integration

The outer SSG framework should use those crates to:

- render pages
- generate IDE-facing artifacts if desired
- bundle CSS and ESM using Rust tooling
- manage watch mode and output layout

## Scope

- This is a component system for static site generation.
- It is not a complete SSG framework.
- It is not an SSR framework.
- It is not a hydration framework.
- It is not a WASM frontend runtime.
- It is not a browser API binding layer for Rust.
- It is not a bundler.
- It is not responsible for watch mode, output directories, or IDE workspace layout.

## Prior Art / Inspirations

The design draws from several existing systems. Each one influences a different part of the proposed architecture, and each one also clarifies what should not be copied.

### Astro

Astro is the main influence on the component semantics.

Relevant influence:

- the distinction between props and slots
- default and named slots
- slot fallback content
- slot forwarding
- static-by-default component behavior
- no implied client runtime just because components exist

What to borrow:

- semantic model for props and slots
- static-first posture

What not to borrow:

- the `.astro` file format
- Astro's broader per-request render-context APIs

### Leptos

Leptos is the main influence on Rust-native component authoring.

Relevant influence:

- `#[component]`-style ergonomics
- strongly typed props
- compile-time checking
- typed children / slot-like composition

What to borrow:

- Rust-first authoring model
- prop ergonomics
- component-as-Rust-function mindset

What not to borrow:

- browser-facing reactive runtime assumptions
- WASM-oriented execution model
- framework-owned client runtime assumptions

### Maud

Maud is the main influence on compile-time HTML authoring ergonomics in Rust.

Relevant influence:

- pleasant `html!`-style authoring
- compile-time template expansion
- small runtime surface

What to borrow:

- ergonomics target for writing static markup
- compile-time-heavy implementation style

What not to borrow:

- exact syntax
- limited notion of components

### markup.rs

`markup.rs` is the main influence on the template-engine core.

Relevant influence:

- compile-time parsing
- reusable render units
- escaping by default
- small runtime surface
- zero-runtime-dependency discipline

What to borrow:

- static rendering discipline
- reusable render abstraction ideas

What not to treat as sufficient:

- `markup.rs` alone does not provide the full component semantics needed here
- explicit props, slots, client artifacts, and style artifacts must be added

### OXC

OXC influences the outer framework and tooling boundary, not the component model itself.

Relevant influence:

- Rust-native JS/TS parser
- resolver
- transformer
- code generation
- minification building blocks

What to borrow:

- the idea that a Rust-only bundling pipeline is viable in the outer framework
- a clean separation between component semantics and bundling/tooling

What to watch out for:

- minification should remain pluggable
- the component crates should not hard-wire themselves to one bundling backend

### The Web Platform

The web platform is the most important target abstraction.

Relevant influence:

- plain HTML
- standard ESM
- DOM events
- ordinary browser APIs

This explains several key decisions:

- no WASM
- no Rust DOM binding layer
- no framework-owned shared runtime
- plain HTML as the default output
- optional future custom-element support rather than mandatory custom elements

## Primary Design Goals

- Components are authored in Rust.
- Compile-time parsing is acceptable.
- Runtime parsing of templates is not acceptable.
- HTML output is deterministic and static-generation-friendly.
- Client behavior, when present, is emitted or referenced as clean ESM.
- No Rust concerns leak into browser output.
- No WASM is involved.
- No framework-owned shared client runtime is required.
- CSS remains a separate concern from the component core.
- The component system is metadata-rich enough for an outer framework to integrate with bundlers, IDE support, and incremental rebuilds.
- The build pipeline can remain Rust-only.
- Authoring should be pleasant and low-boilerplate.

## Non-Goals

- No mandatory file layout.
- No mandatory sidecar files.
- No mandatory custom elements.
- No mandatory Shadow DOM.
- No mandatory client-side state/store runtime.
- No mandatory Node.js, Deno, Bun, or Vite involvement.
- No direct `.d.ts` file emission by the component system itself.
- No direct CSS or ESM bundling by the component system itself.
- No custom import-resolution system owned by the component system.

## High-Level Architecture

There should be three core crates.

### `components-core`

Responsibilities:

- define component semantics
- define render traits and HTML output types
- define props and slots abstractions
- define render-time tracking of component usage
- define metadata types for components and associated artifacts
- define client-facing serialization boundaries
- define abstract artifact descriptors for style, client, and declaration outputs
- define stable identity and marker generation rules

This crate should not:

- read or write project files as part of normal operation
- perform bundling
- parse CSS or JS/TS
- own watch mode
- decide IDE layout

### `components-macros`

Responsibilities:

- provide proc macros for ergonomic component definitions
- optionally provide markup-authoring macros
- generate Rust rendering code
- generate metadata accessors
- generate compile-time prop and slot validation
- generate render-context recording code

This crate should not:

- emit files
- resolve imports
- enforce a filesystem convention
- perform bundling or minification

### `components-build`

Responsibilities:

- expose build-facing APIs for frameworks
- convert raw or framework-resolved component metadata into declaration models
- convert page render data into page manifests
- convert page manifests into framework-neutral client attachment planning data
- expose invalidation-relevant dependency and fingerprint data
- provide helper APIs for frameworks that want to materialize IDE-facing files

This crate should not:

- require writing `.d.ts` to disk
- own directory conventions
- own watch mode
- own bundling
- assume a specific framework layout

### Optional later crates

These are optional and should not shape the first implementation:

- `components-testing`
- `components-devtools`
- `components-css-*` adapters
- framework-specific integration crates

## Core Conceptual Model

A component is a Rust-defined render unit with:

- typed props
- typed slots
- HTML output
- optional source-declared artifact hints
- optional framework-resolved client and style associations

The component system should separate:

- render-time concerns
- metadata and artifact concerns
- framework policy concerns

This separation is the main architectural boundary to protect.

## Authoring Model

The component system should support two Rust authoring styles:

- a macro-assisted style for pleasant markup authoring
- a pure Rust builder style for users who want no template DSL

The system should not require any particular directory structure.

A framework may choose to support a convention such as:

- a component source file
- an optional client sidecar
- an optional style sidecar

But the component system itself should work with a more abstract model:

- a component may have zero or more associated artifacts
- those artifacts may come from files, generated content, or virtual sources
- the framework decides how those artifacts are authored and materialized

This means the component system should model artifact association abstractly, not path-first.

The framework may also associate client, style, or declaration artifacts entirely externally. In other words, a component does not need to name or own its associated artifacts in source for the framework to connect them later.

## Metadata Resolution Lifecycle

The component system needs two distinct metadata layers.

The first layer is raw component metadata, discovered from Rust component definitions. This includes:

- component identity
- prop schema
- slot schema
- ref schema
- optional source-declared artifact hints

The second layer is framework-resolved component metadata, produced after the outer framework applies its own policy. This includes:

- associated client artifacts
- associated style artifacts
- associated declaration artifacts
- marker requirements implied by those associations

This distinction matters because artifact association may be:

- source-declared
- convention-based
- configured externally by the framework
- generated elsewhere in the build graph

The intended lifecycle is:

1. discover raw component metadata
2. let the outer framework resolve artifact associations and marker requirements
3. render using framework-resolved component descriptors
4. produce page manifests and client-attachment planning data from that resolved view

Rendering and instance recording should operate on the resolved view, not only on raw `ComponentMeta`, because marker emission and client/style requirements depend on framework policy.

## Component Semantics

### Props

Requirements:

- props are strongly typed Rust inputs
- required and optional props are supported
- defaults are supported
- compile-time validation should catch unknown props and wrong types
- prop metadata should be available to frameworks

Important distinction:

- some props exist only at SSG render time
- some props may be exposed to client artifacts
- client-exposed props must be explicitly modeled and serializable

The system should not assume all props automatically cross the Rust-to-client boundary.

### Slots

Requirements:

- support a default slot
- support named slots
- support fallback content
- support forwarding
- slots are typed as renderable content, not strings

The slot model should take inspiration from Astro's semantics while remaining Rust-native in its typing.

### Phase-1 Slot Representation

For phase 1, props and slots should remain distinct in the public component model, in diagnostics, and in metadata.

That means:

- props are typed configuration inputs
- slots are typed projected child content
- `.d.ts` generation and other tooling should derive from that semantic distinction

However, macro expansion may still lower a component internally to a single generated component-input struct if that simplifies implementation. If it does, that lowered struct is a private implementation detail and should not become the conceptual API.

So the intended rule is:

- public semantics: props and slots are distinct
- internal lowering: a generated private input struct may carry both
- metadata and tooling: props and slots remain modeled separately

### Rendering

Requirements:

- components render to plain HTML by default
- multiple roots should be allowed if the output model supports them
- static components should not carry unnecessary markers
- client-enabled components may require stable markers
- style scoping may require stable markers
- marker emission must be deterministic

The default output should remain semantic HTML, not custom elements.

## Client Behavior Model

This is the most important design constraint.

The component system should not model browser APIs in Rust. It should not provide a generalized Rust DOM manipulation layer. It should not try to express every possible browser intent through Rust abstractions.

Instead, the component system should do only this:

- record that a component has an associated client artifact
- record which DOM anchors or refs the client artifact expects
- record which serializable data is exposed to that client artifact
- record enough instance information during page render for the framework to wire client behavior later

The browser-facing behavior itself should remain ordinary ESM.

This keeps the architecture clean:

- Rust owns component structure and metadata
- ESM owns actual browser APIs
- no WebIDL-generated Rust browser layer is needed

This is also the intended interop model for third-party JavaScript libraries and browser APIs in general: client code should be able to import ordinary ESM and call browser APIs directly, without any Rust-specific bridge layer.

## No Shared Framework Runtime

The system should not ship a mandatory shared runtime for stores, signals, or component coordination.

Cross-component communication on the client should be possible through:

- DOM events
- user-authored shared ESM modules
- third-party ESM libraries if the site author wants them

This preserves a major advantage of Astro-like architectures: no client runtime is imposed just because components exist.

## One Component Definition, One Client Module

A critical rule:

- one component definition may correspond to one client artifact
- many instances of that component on a page must share that same client artifact
- no per-instance ESM modules should be emitted
- no per-instance CSS should be emitted

Per-instance variation belongs in serialized data, not in duplicated source output.

## Style Model

CSS should remain outside the component core, but the component core must describe enough information for CSS integration.

The component system should support:

- optional style artifact hints and/or resolved style artifact metadata
- stable scope identities or scoping hints
- optional HTML marker requirements for scoping

The component system should not:

- enforce plain CSS or CSS-in-Rust
- perform CSS bundling
- perform CSS minification
- hard-wire a single scoping strategy
- require Shadow DOM

A framework or companion CSS layer may choose different strategies, including:

- plain external CSS
- CSS-in-Rust-generated output
- `@scope`
- attribute-based scoping
- class-based scoping

The component-system crates should remain neutral.

## Plain HTML vs Custom Elements

Default position:

- components render plain HTML
- custom elements are not the default
- Shadow DOM is not the default

Reasoning:

- plain HTML is better for static output
- plain HTML is simpler to style and integrate
- plain HTML avoids upgrade timing and Shadow DOM complexity
- custom elements and Shadow DOM should be optional future features, not baseline assumptions

If custom-element support is ever added, it should be:

- opt-in
- per component
- modeled as alternate metadata and framework behavior
- never required for the whole ecosystem

## Artifact Model

The artifact model should be abstract and framework-facing.

The component system should talk about artifact identities and artifact descriptions, not mandatory file paths.

Suggested artifact kinds:

- client artifact
- style artifact
- declaration artifact
- framework bootstrap artifact

Suggested origin kinds:

- file-backed
- generated
- virtual

Meaning:

- file-backed: an artifact comes from an existing file on disk
- generated: an artifact can be materialized as deterministic source by the framework or another crate when needed
- virtual: an artifact exists only as a logical or in-memory build input and may never need to exist as a stable file on disk

Suggested common artifact fields:

- stable artifact ID
- artifact kind
- origin kind
- optional source locations
- logical module name if relevant
- dependency hints
- framework-facing content-generation handle or descriptor

This lets the same component system support:

- sidecar files authored by users
- generated ESM/CSS produced elsewhere in the framework
- IDE-friendly generated outputs
- purely in-memory pipelines

## Identifier Stability

The docs currently use the word "stable" for several identifiers. That stability should be defined per category rather than assumed to mean one thing everywhere.

### Component IDs

Component IDs should be stable across:

- process restarts
- rebuilds of unchanged source

Initially, they do not need to be stable across:

- source moves
- renames
- deliberate identity-affecting refactors

### Artifact IDs

Artifact IDs should be stable across:

- process restarts
- rebuilds where the same resolved artifact association still applies

They do not need to be stable across:

- framework policy changes
- changes in artifact origin
- changes in association rules

### Scope IDs

Scope IDs should be stable across:

- rebuilds of unchanged component and style-association inputs

They do not need to be stable across:

- changes in CSS scoping policy
- changes in associated style artifacts

### Instance IDs

Instance IDs are page-local and render-local.

They should be stable within one render and deterministic for the same rendered page shape, but they should not be treated as long-lived identifiers across arbitrary template edits or tree reordering.

### Marker Identities

Marker identities should be deterministic for one rendered page and sufficiently stable for client attachment on unchanged output, but they should not be treated as a public compatibility contract across unrelated markup changes.

### Fingerprints

Fingerprints are invalidation tokens, not semantic identifiers.

They should be stable for identical logical inputs, but frameworks should treat them as cache keys rather than as user-visible IDs.

## Core Metadata Types

The exact API can evolve, but the system needs types equivalent to the following.

### `ComponentMeta`

Should include:

- stable component ID
- human-readable component name
- Rust module path or equivalent identity
- source span or source location if available
- prop schema
- slot schema
- ref schema
- optional source-declared artifact hints
- fingerprint inputs

### `ResolvedComponentDescriptor`

Should include:

- a reference to the raw `ComponentMeta`
- the resolved client artifact descriptor if any
- the resolved style artifact descriptor if any
- the resolved declaration descriptor if any
- marker requirements implied by the resolved associations

### `PropMeta`

Should include:

- prop name
- type identity or display form
- whether required
- whether defaulted
- whether client-exposed
- client-facing type info
- serialization requirements

### `SlotMeta`

Should include:

- slot name
- whether default slot
- whether required
- multiplicity or fragment support
- fallback presence if needed for tooling

### `RefMeta`

Should include:

- ref name
- whether root or descendant
- whether required for client behavior
- optional DOM kind hint later if useful

### `ClientArtifactMeta`

Should include:

- artifact ID
- origin kind
- optional source location
- logical artifact handle or framework-facing reference
- expected mount contract version
- client-exposed prop schema
- required refs
- whether marker injection is required

### `StyleArtifactMeta`

Should include:

- artifact ID
- origin kind
- optional source location
- scoping preference
- stable scope identity
- whether HTML scope markers are required

### `DeclarationModel`

Should include enough information for a framework to generate TypeScript declaration content for client-facing contracts, including:

- client prop types
- ref types
- mount context type
- shared helper types if needed

Important rule:

- the component system produces declaration data or declaration renderers
- the framework decides whether to emit `.d.ts` files, where to place them, and whether to expose them to IDEs
- declaration generation should follow semantic metadata for props and slots, not any private lowered implementation struct used internally by macro expansion

## Render-Time Tracking Types

The component system also needs page-level runtime-independent tracking.

### `RenderContext`

Responsibilities:

- accumulate used component IDs
- accumulate client instance records
- accumulate style requirements
- accumulate marker usage
- keep page-level deterministic ordering if needed

### `InstanceRecord`

Should include:

- component ID
- stable page-local instance ID
- resolved client artifact ID if any
- serialized client payload
- marker identities
- ref marker mapping
- style scope marker mapping if applicable

### `PageManifest`

Should include:

- page identity
- used component set
- used client artifact set
- used style artifact set
- instance records
- client attachment planning inputs
- invalidation or fingerprint data

This manifest is a primary integration point for the outer framework.

## How the Outer SSG Framework Should Integrate

The outer framework should treat the component system as a semantic engine and metadata provider.

### 1. Component Discovery

The framework should compile the user's Rust code and enumerate available component metadata through APIs exposed by the component system.

The component system should make this possible through generated metadata accessors or a registry mechanism, but the framework should own:

- when discovery runs
- how results are cached
- how invalidation is tracked

### 2. Resolving Framework Policy

Before rendering, the framework should resolve raw component metadata into framework-resolved component descriptors.

This step is where the framework may:

- associate client artifacts
- associate style artifacts
- associate declaration artifacts
- decide which markers are required

This step may use:

- source-declared hints
- file conventions
- generated artifacts
- configuration or framework policy

### 3. Rendering Pages

The framework should render pages using `components-core` rendering APIs and the framework-resolved component descriptors.

During render:

- `RenderContext` should collect page-level component usage
- `RenderContext` should collect client instance records
- `RenderContext` should collect style requirements

After render:

- HTML output is available
- `PageManifest` is available
- framework can proceed to artifact planning

### 4. Generating IDE-Facing Artifacts

If the framework wants IDE support for client-side code, it may choose to materialize declaration data as `.d.ts` files.

That is optional and framework-owned.

The framework may choose:

- to emit files into a generated directory
- to build a conventional sidecar layout for users
- to maintain stable import paths for editor tooling
- not to emit these files at all in some modes

The component system should not assume any of these choices.

Those declarations may be derived from raw semantic metadata, from framework-resolved component descriptors, or from both, depending on how the framework models client-side contracts.

### 5. Generating Page Bootstrap Artifacts

From `PageManifest` and the framework-resolved component descriptors, the framework should generate page-level client bootstrap artifacts.

Important rule:

- page bootstrap generation belongs to the framework
- the component system should only expose framework-neutral client attachment planning data

The resulting bootstrap should:

- import each used component client artifact once
- locate all instances for each component on the page
- reconstruct mount contexts from HTML markers and serialized data
- call the component client contract for each instance

### 6. Handling CSS

The framework should decide how to turn style artifact metadata into bundled CSS output.

Depending on framework policy, this may involve:

- direct concatenation or planning
- a CSS processor implemented in Rust
- CSS-in-Rust adapters
- scope handling using marker metadata

The component system should stay neutral and simply expose what is needed.

### 7. Bundling and Tree-Shaking

This belongs entirely to the outer framework.

The framework should take:

- framework-generated bootstrap or client-entry artifacts
- client artifacts
- style artifacts
- any generated declaration or manifest data it needs

and run them through Rust-native tooling for:

- module resolution
- TS lowering if needed
- tree-shaking
- chunking
- minification
- final output assembly

The component system should not know which bundler implementation is used.

### 8. Watch Mode

Watch mode belongs entirely to the outer framework.

If the framework supports file-based authoring conventions, it should decide what to watch, such as:

- Rust component sources
- authored client or style sidecars
- generated frontend sources
- frontend dependency graphs discovered by its bundler layer

The component system may expose dependency hints and fingerprints, but it should not own watch orchestration.

## Optional Framework Convention: File-Based Co-Location

A framework may choose to provide a convention such as:

- one directory per component
- a Rust component source
- optional `client.ts`
- optional `style.css`

This is a framework feature, not a component-system requirement.

If a framework chooses this convention, it should document:

- how sidecars are discovered
- where generated declarations go
- what import paths are stable for IDEs
- how generated files are cleaned and regenerated

But the underlying component system must remain usable without this convention.

## Generated Declarations

The component system should expose a declaration-generation API or declaration model, not physical files.

The framework may then choose to generate TypeScript types such as:

- client-visible prop interfaces
- ref interfaces
- mount context interfaces

Example conceptual output, if a framework chooses to materialize it:

```ts
export interface ButtonClientProps {
  variant: "primary" | "secondary";
  disabled?: boolean;
}

export interface ButtonRefs {
  root: HTMLElement;
}

export interface ButtonMountContext {
  root: HTMLElement;
  refs: ButtonRefs;
  props: ButtonClientProps;
}
```

Again, this is framework policy. The component system should only provide the structured data needed to derive it.

If the framework chooses to support IDE-friendly TypeScript authoring, it may also provide a `tsconfig.json` or equivalent editor-facing configuration purely for tooling. That does not imply that the build itself depends on Node.js or TypeScript tooling.

## Import Resolution and Tooling Expectations

The component system should not invent a custom module resolver for client or style artifacts.

The intended model is:

- JS and TS use ordinary ESM import semantics
- CSS uses ordinary CSS import semantics
- package imports and relative imports remain standard
- any IDE-facing resolution support is provided by the outer framework using normal editor conventions

This should hold whether artifacts are:

- file-backed
- generated
- virtual

The goal is that editor support and bundling can rely on standard expectations even if the framework chooses to generate some of the sources or declarations.

## Suggested Public API Shape

This is conceptual, not final.

`components-core` should likely expose:

- a `Component` trait or generated equivalent
- `RenderContext`
- `HtmlFragment` or equivalent
- slot and renderable abstractions
- metadata types
- serialization helpers for client payloads

`components-macros` should likely expose:

- `#[component]`
- optional markup macro for ergonomic HTML authoring
- annotations for prop defaults and client exposure if needed

`components-build` should likely expose:

- metadata enumeration helpers
- declaration-model generation
- page-manifest to bootstrap-plan conversion
- invalidation or fingerprint helpers

## Implementation Plan

### Phase 1: Core Rendering and Semantics

Deliverables:

- HTML output abstraction
- typed props
- typed slots
- render context
- component metadata model
- basic proc-macro component definition support
- deterministic HTML rendering

Acceptance criteria:

- a static component renders deterministic HTML
- props are compile-time checked
- slots support default and named forms
- no runtime template parsing exists
- metadata can be queried by a framework

### Phase 2: Artifact Metadata

Deliverables:

- abstract client, style, and declaration artifact descriptors
- ref schema support
- marker requirement modeling
- client-exposed prop schema support

Acceptance criteria:

- a component can declare or expose an associated client artifact abstractly
- a component can declare or expose an associated style artifact abstractly
- metadata is not path-first or sidecar-file-dependent
- the same model works for file-backed and generated artifacts

### Phase 3: Render-Time Usage Tracking

Deliverables:

- instance records
- page manifests
- style requirement tracking
- client instance payload tracking

Acceptance criteria:

- rendering a page yields HTML plus a `PageManifest`
- many instances of the same component share the same client artifact identity
- instance-specific data is recorded separately from module-level artifacts
- purely static components do not force client metadata into the page

### Phase 4: Declaration Models

Deliverables:

- framework-facing declaration model APIs
- TS-facing mount context model
- client prop and ref type derivation

Acceptance criteria:

- a framework can derive `.d.ts` content from component metadata
- the component system does not itself require file emission
- declaration output changes when client-facing component contracts change

### Phase 5: Bootstrap Planning APIs

Deliverables:

- page-manifest to framework-neutral client attachment planning data
- stable instance lookup strategy
- framework-facing mount contract spec

Acceptance criteria:

- a framework can generate one page bootstrap or client-entry artifact per page from the planning data
- the bootstrap imports each used client artifact once
- no per-instance ESM output is required
- the mount contract stays plain ESM-oriented

### Phase 6: Ergonomics and Diagnostics

Deliverables:

- improved macro diagnostics
- improved span reporting
- lower-boilerplate prop and slot declaration
- better debug and display tools for metadata and manifests

Acceptance criteria:

- common components are pleasant to write
- compile-time errors point to user code
- the mental model of props, slots, client artifacts, and style artifacts is consistent

## Acceptance Criteria Consolidated

These are the design-level acceptance criteria implied by this discussion.

- The component system is library-only and reusable.
- The outer SSG framework owns build orchestration.
- Components are authored in Rust.
- Compile-time parsing is allowed.
- Runtime template parsing is not allowed.
- HTML output is deterministic and SSG-friendly.
- Browser output contains no WASM.
- Browser output contains no Rust-specific runtime concerns.
- Client behavior is modeled as ordinary ESM integration.
- The component system does not model the browser DOM in Rust.
- The component system does not require a shared client runtime.
- Cross-component communication can use DOM events or user-owned ESM modules.
- CSS remains separate from the component core.
- Plain HTML is the default render target.
- Custom elements and Shadow DOM are optional future features, not baseline requirements.
- One component definition corresponds to one client artifact and one style artifact set.
- Multiple instances of a component share those artifacts.
- Per-instance variability is represented as serialized data, not duplicated modules.
- The component system does not require any directory layout.
- A framework may provide a directory convention, but that is policy, not core semantics.
- The component system does not emit `.d.ts` files directly as a core responsibility.
- The framework may choose to materialize IDE-facing declaration files from declaration data.
- The component system exposes enough metadata for frameworks to support bundling, tree-shaking, IDE integration, and incremental rebuilds.
- Watch mode belongs to the outer framework, not the component system.

## Do and Don't Summary

Do:

- keep the component crates focused on semantics and metadata
- keep browser behavior in ordinary ESM
- keep CSS abstract and pluggable
- keep output deterministic
- keep the framework boundary explicit
- keep file layout and generated files as framework policy

Don't:

- build a Rust browser API layer
- build a mandatory shared runtime
- emit per-instance client modules
- force a sidecar-file convention
- force `.d.ts` file emission in the component crates
- let the component system turn into a full framework or bundler

## Recommended Next Step

The next artifact to write should be a concrete API sketch for:

- `Component`
- `RenderContext`
- `ComponentMeta`
- `ClientArtifactMeta`
- `StyleArtifactMeta`
- `DeclarationModel`
- `PageManifest`
- bootstrap-plan generation inputs and outputs

That would turn this architecture from a planning document into an executable implementation target.
