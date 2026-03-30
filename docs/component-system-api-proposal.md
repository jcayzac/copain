# Rust Component System API Proposal

## Status

This document is a proposal, not a definitive plan.

It sketches a possible API surface for the component-system crates described in [component-system-design.md](/Users/julien.cayzac/github/jcayzac/copain/docs/component-system-design.md). Names, trait shapes, module layout, and data-model details are all provisional.

The goal of this document is to make the architecture concrete enough to evaluate implementation strategy, crate boundaries, and framework integration.

## Purpose

This proposal covers:

- candidate Rust APIs for `components-core`
- candidate proc-macro behavior for `components-macros`
- candidate build-facing APIs for `components-build`
- candidate data contracts between the component system and the outer SSG framework

This proposal does not define:

- a final stable public API
- a final macro syntax
- a final registry mechanism
- a final bundling implementation
- a final CSS integration model

## Design Constraints Carried Into This Proposal

- Components are authored in Rust.
- Compile-time parsing is acceptable.
- Runtime template parsing is not acceptable.
- Rendering is SSG-oriented and deterministic.
- Plain HTML is the default output.
- Client behavior is represented as ordinary ESM integration.
- No WASM is involved.
- No Rust DOM binding layer exists.
- No framework-owned shared client runtime exists.
- CSS is a separate concern.
- The component system is library-only and reusable.
- The outer SSG framework owns output layout, watch mode, generated files, and bundling.
- The component system does not own custom module resolution for JS, TS, or CSS.

## Proposed Crate Responsibilities

### `components-core`

Proposed responsibilities:

- rendering primitives
- render context
- component trait contracts
- props/slots/renderable abstractions
- metadata types
- artifact descriptor types
- page manifest types
- client-payload serialization helpers

### `components-macros`

Proposed responsibilities:

- `#[component]`
- optional markup macro
- compile-time prop/slot validation
- metadata generation
- registration hooks
- render-context instrumentation

### `components-build`

Proposed responsibilities:

- declaration-model generation
- bootstrap-plan generation
- metadata enumeration helpers
- invalidation/fingerprint helpers
- framework-facing render outputs and plans

## Proposed `components-core` API

### Render Output

The core crate needs a rendering abstraction that is simple, deterministic, and not tied to a specific templating syntax.

Proposed types:

```rust
pub struct HtmlFragment {
    // Opaque for now.
}

pub trait IntoHtml {
    fn into_html(self) -> HtmlFragment;
}

pub trait Render {
    fn render(self, cx: &mut RenderContext) -> HtmlFragment;
}
```

Notes:

- `HtmlFragment` should be opaque initially so internals can evolve.
- `Render` is the core boundary. A component ultimately produces HTML through it.
- `IntoHtml` is useful for composability in builder APIs and slot values.

### Components

A component should feel like a strongly typed Rust function, but frameworks need runtime-independent metadata as well.

Candidate shape:

```rust
pub trait Component {
    type Props;

    fn meta() -> &'static ComponentMeta;
    fn render(props: Self::Props, cx: &mut RenderContext) -> HtmlFragment;
}
```

This may end up being generated rather than directly user-implemented.

Alternative shape:

```rust
pub trait Component<Props> {
    fn meta() -> &'static ComponentMeta;
    fn render(props: Props, cx: &mut RenderContext) -> HtmlFragment;
}
```

Current bias:

- prefer generated implementations from `#[component]`
- do not require users to manually implement these traits

### Render Context

The render context is the main accumulation point during page generation.

Candidate API:

```rust
pub struct RenderContext {
    // internal state
}

impl RenderContext {
    pub fn new() -> Self;

    pub fn register_component_use(&mut self, component: &'static ComponentMeta);

    pub fn register_style_use(&mut self, style: &'static StyleArtifactMeta);

    pub fn register_client_instance(&mut self, instance: InstanceRecord);

    pub fn alloc_instance_id(&mut self, component_id: ComponentId) -> InstanceId;

    pub fn finish(self, page: PageIdentity) -> PageManifest;
}
```

Notes:

- `RenderContext` is not a browser runtime abstraction.
- It exists only during SSG rendering.
- It should be deterministic and side-effect-free except for manifest accumulation.

### Props

Props are normal Rust inputs, but the framework needs metadata and possibly client-facing type information.

Candidate metadata:

```rust
pub struct PropMeta {
    pub name: &'static str,
    pub rust_type: TypeRepr,
    pub required: bool,
    pub has_default: bool,
    pub client_exposure: ClientExposure,
}

pub enum ClientExposure {
    Hidden,
    Serializable { ts_type: TsTypeRepr },
}
```

Candidate helper traits:

```rust
pub trait SerializeForClient {
    fn serialize_for_client(&self) -> Result<ClientValue, ClientSerializeError>;
}
```

Notes:

- not every prop should be client-visible
- client visibility should be explicit
- failure to serialize client-exposed data should be a clear error
- client-visible data exists to support ordinary ESM interop, including direct use of browser APIs and third-party ESM libraries from client code

### Slots

Slots should support default and named semantics, with typing kept Rust-native.

Candidate shape:

```rust
pub trait SlotValue: IntoHtml {}

pub struct Slots {
    // internal
}

impl Slots {
    pub fn has(&self, name: SlotName) -> bool;
    pub fn get(&self, name: SlotName) -> Option<&dyn SlotValue>;
}

pub struct SlotMeta {
    pub name: SlotName,
    pub is_default: bool,
    pub required: bool,
    pub multiple: bool,
    pub has_fallback: bool,
}
```

Possible user-facing models:

- slots as generated props fields
- slots as a dedicated generated struct

Current bias:

- generated props struct should probably include generated slot fields
- metadata should still keep slots explicit for tooling

### Refs and Attachment Markers

The component system should not expose DOM handles in Rust. It should only describe attachment points and refs expected by client artifacts.

Candidate metadata:

```rust
pub struct RefMeta {
    pub name: &'static str,
    pub kind: RefKind,
    pub required: bool,
}

pub enum RefKind {
    Root,
    Descendant,
}
```

Candidate render-time helpers:

```rust
pub struct MarkerPlan {
    pub root_marker: Option<MarkerName>,
    pub refs: Vec<RefMarker>,
    pub scope_marker: Option<MarkerName>,
}
```

Notes:

- markers should be emitted only when needed
- refs are for framework/client integration, not for Rust-side DOM logic

### Artifacts

Artifacts should be abstract. They should not require real files.

Candidate shared types:

```rust
pub struct ArtifactId(pub u64);

pub enum ArtifactKind {
    Client,
    Style,
    Declaration,
    PageBootstrap,
}

pub enum ArtifactOrigin {
    File { path: std::path::PathBuf },
    Generated { logical_name: String },
    Virtual { debug_name: String },
}
```

Meaning of these origin kinds:

- `File` means the artifact comes from an existing file on disk.
- `Generated` means the framework or another crate can materialize deterministic source for it when needed.
- `Virtual` means the artifact exists only as a logical or in-memory build input and may never need to exist as a stable file on disk.

Client artifact metadata:

```rust
pub struct ClientArtifactMeta {
    pub id: ArtifactId,
    pub origin: ArtifactOrigin,
    pub export_name: Option<String>,
    pub props: Vec<PropMeta>,
    pub refs: Vec<RefMeta>,
    pub mount_contract: MountContractVersion,
    pub requires_markers: bool,
}
```

Style artifact metadata:

```rust
pub struct StyleArtifactMeta {
    pub id: ArtifactId,
    pub origin: ArtifactOrigin,
    pub scoping: ScopingPreference,
    pub scope_id: Option<ScopeId>,
    pub requires_markers: bool,
}
```

Declaration artifact metadata:

```rust
pub struct DeclarationArtifactMeta {
    pub id: ArtifactId,
    pub logical_name: String,
}
```

### Component Metadata

The outer framework needs a stable description of each component definition.

Candidate shape:

```rust
pub struct ComponentMeta {
    pub id: ComponentId,
    pub name: &'static str,
    pub rust_path: &'static str,
    pub props: &'static [PropMeta],
    pub slots: &'static [SlotMeta],
    pub refs: &'static [RefMeta],
    pub client: Option<&'static ClientArtifactMeta>,
    pub style: Option<&'static StyleArtifactMeta>,
    pub declarations: Option<&'static DeclarationArtifactMeta>,
    pub markers: MarkerRequirements,
    pub fingerprint: Fingerprint,
}
```

Notes:

- this type should be cheap to store and query
- most fields should likely be `'static`
- the framework should be able to enumerate these metas without instantiating components

### Render-Time Records

The framework needs page-level usage data after render.

Candidate types:

```rust
pub struct InstanceRecord {
    pub component_id: ComponentId,
    pub instance_id: InstanceId,
    pub client_artifact_id: Option<ArtifactId>,
    pub props: Option<ClientValue>,
    pub markers: InstanceMarkers,
}

pub struct PageManifest {
    pub page: PageIdentity,
    pub components: Vec<ComponentId>,
    pub client_artifacts: Vec<ArtifactId>,
    pub style_artifacts: Vec<ArtifactId>,
    pub instances: Vec<InstanceRecord>,
    pub fingerprint: Fingerprint,
}
```

This is the core framework integration output from a render.

## Proposed `components-macros` API

### `#[component]`

This proposal assumes a `#[component]` proc macro is the primary ergonomic entry point.

Conceptual usage:

```rust
#[component]
pub fn Button(
    label: String,
    #[prop(default = ButtonVariant::Primary)]
    variant: ButtonVariant,
    children: DefaultSlot,
) -> impl IntoHtml {
    // ...
}
```

Proposed macro responsibilities:

- generate a props struct if needed
- generate metadata statics
- generate `Component` integration
- validate prop defaults
- validate slot declarations
- instrument render-time manifest recording

### Optional Markup Macro

This proposal leaves room for a markup macro, but does not make its syntax definitive.

Conceptual placeholder:

```rust
html! {
    button class="button" {
        {label}
        {children}
    }
}
```

This should remain explicitly provisional. The first implementation could use a very small macro surface or even builder-only internals until the semantics are stable.

### Client Exposure Annotations

The macro layer likely needs explicit annotations for which props are visible to the client contract.

Possible direction:

```rust
#[component]
pub fn Counter(
    #[client]
    initial: i32,
    step: i32,
) -> impl IntoHtml {
    // ...
}
```

This is only a proposal. The final design may prefer a different mechanism such as:

- an attribute on props
- an attribute on the component
- an explicit associated client-contract block

### Artifact Association Annotations

Because artifact association must support file-backed, generated, and virtual sources, the macro should not force a single sidecar convention.

Possible directions:

```rust
#[component]
#[client_artifact(source = "convention")]
#[style_artifact(source = "convention")]
pub fn Button(...) -> impl IntoHtml { ... }
```

or

```rust
#[component]
#[client_artifact(logical = "button")]
#[style_artifact(logical = "button")]
pub fn Button(...) -> impl IntoHtml { ... }
```

or no attribute at all, with the outer framework associating artifacts later.

Current bias:

- keep artifact association flexible
- do not hard-wire the macro API to sidecar file paths too early
- allow the outer framework to associate client or style artifacts externally, without requiring the component source to name them directly

## Proposed `components-build` API

This crate is where framework-facing planning APIs should live.

### Metadata Enumeration

Conceptual API:

```rust
pub fn all_components() -> Vec<&'static ComponentMeta>;
```

This may be backed by:

- generated registration tables
- `inventory`
- explicit framework registration
- another mechanism

This proposal intentionally does not lock that down yet.

### Declaration Models

The component system should expose declaration data, not force file emission.

Candidate types:

```rust
pub struct DeclarationModel {
    pub logical_name: String,
    pub exports: Vec<TsExport>,
}

pub enum TsExport {
    Interface(TsInterface),
    TypeAlias(TsTypeAlias),
}
```

Candidate API:

```rust
pub fn declaration_model(component: &ComponentMeta) -> Option<DeclarationModel>;

pub fn render_typescript_declaration(model: &DeclarationModel) -> String;
```

Notes:

- a framework may use the structured form directly
- a framework may also render it to `.d.ts`
- the component system should support both

### Bootstrap Planning

The framework needs a plan for generating page-level ESM entrypoints.

Candidate types:

```rust
pub struct BootstrapPlan {
    pub page: PageIdentity,
    pub imports: Vec<BootstrapImport>,
    pub mount_groups: Vec<MountGroup>,
}

pub struct BootstrapImport {
    pub artifact_id: ArtifactId,
    pub specifier: String,
    pub export_name: Option<String>,
}

pub struct MountGroup {
    pub component_id: ComponentId,
    pub artifact_id: ArtifactId,
    pub instances: Vec<InstanceRecord>,
}
```

Candidate API:

```rust
pub fn bootstrap_plan(
    manifest: &PageManifest,
    components: &ComponentIndex,
) -> BootstrapPlan;
```

The framework can then decide whether to:

- emit a real `.ts` file
- emit a generated `.js` file
- keep the plan in memory and hand it directly to a bundler layer

### Invalidation Helpers

Candidate types:

```rust
pub struct InvalidationKey(pub u64);

pub struct ComponentDependencyHint {
    pub component_id: ComponentId,
    pub sources: Vec<DependencySource>,
}
```

Candidate API:

```rust
pub fn component_invalidation_key(component: &ComponentMeta) -> InvalidationKey;

pub fn page_invalidation_key(manifest: &PageManifest) -> InvalidationKey;
```

These are framework-oriented helpers, not a watch subsystem.

## Proposed Framework Integration Flow

This is still a proposal, but it is the most likely end-to-end flow.

### 1. Compile and Discover

The framework:

- builds the Rust code
- enumerates `ComponentMeta`
- caches component metadata and invalidation keys

### 2. Optionally Materialize IDE Support

The framework may:

- derive `DeclarationModel`
- render `.d.ts` files
- arrange a TS-friendly generated directory
- provide a `tsconfig.json` or equivalent editor-facing configuration

or it may skip this entirely.

The component system should not care.

This editor-facing support is separate from the build pipeline. The build may remain entirely Rust-only even if the framework generates TypeScript declarations for IDE use.

### 3. Render Pages

The framework:

- creates a `RenderContext`
- renders a page
- obtains HTML plus `PageManifest`

### 4. Plan Client and Style Outputs

From `PageManifest`, the framework:

- determines which client artifacts are used
- determines which style artifacts are used
- builds a `BootstrapPlan`

### 5. Bundle in the Outer Layer

The framework then:

- resolves client/style artifact sources
- generates page entrypoints if desired
- bundles and tree-shakes with Rust tooling
- emits final assets

The component system ends before this step.

Importantly, resolution here should follow ordinary ESM and CSS expectations. The component system should not require a custom resolver protocol.

## Example: Static Component

Conceptual user code:

```rust
#[component]
pub fn Card(
    title: String,
    body: DefaultSlot,
) -> impl IntoHtml {
    html! {
        article class="card" {
            h2 { {title} }
            div class="card-body" {
                {body}
            }
        }
    }
}
```

Expected behavior:

- no client artifact
- optional style artifact if framework associates one
- no extra markers unless style scoping requires them
- page manifest records component and style usage only

## Example: Client-Enabled Component

Conceptual user code:

```rust
#[component]
pub fn Counter(
    #[client]
    initial: i32,
) -> impl IntoHtml {
    html! {
        button data_ref="root" {
            {initial}
        }
    }
}
```

Conceptual metadata outcome:

- `ComponentMeta` has a `ClientArtifactMeta`
- `initial` is marked client-exposed
- `root` is a required ref or root marker

Conceptual render outcome:

- the page HTML includes stable markers for the component instance
- `RenderContext` records one `InstanceRecord` per rendered instance
- the framework later groups all such instances under one client artifact in the bootstrap plan

## Open Questions

These points are intentionally unresolved in this proposal.

- Should `Component` be a real public trait, or only a generated internal contract?
- Should slots be modeled as generated props fields, or as a dedicated `Slots` input?
- How much of the props API should be driven by generated props structs versus function parameters?
- What registry/discovery mechanism is the least awkward for frameworks?
- How much source-span information should be retained in metadata?
- Should declaration models be generic over a TS AST representation, or use a crate-local minimal IR?
- What is the smallest useful initial markup macro surface?
- How should client-artifact association be represented when the framework, not the component, owns that policy?

## Current Recommendation

The first implementation should optimize for:

- a small `components-core`
- a flexible `components-build`
- a minimal but ergonomic `#[component]`
- a strongly typed metadata model
- a render context that cleanly produces `PageManifest`

It should avoid prematurely freezing:

- exact macro syntax
- exact registration strategy
- exact `.d.ts` rendering model
- exact client-artifact association syntax
- exact CSS scoping strategy

## Acceptance Criteria for This Proposal

This API proposal is useful if it is sufficient to:

- begin implementing the three-crate split
- write a prototype `#[component]`
- render a page into HTML plus `PageManifest`
- derive a declaration model for client-facing props and refs
- derive a bootstrap plan from a page manifest
- keep the outer framework in charge of files, watch mode, bundling, and IDE layout

If it cannot support those tasks, it is too vague. If it hard-wires too many policy decisions into the component crates, it is too rigid.
