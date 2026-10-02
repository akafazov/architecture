# ADR 015: Platform Mesh Installation

| Status          | Proposed          |
|-----------------|-------------------|
| Date            | 2026-10-02        |
| Decision-makers | Platform Mesh     |

## Context and Problem Statement

Until recently, Platform Mesh provided only a local installation path (local-setup) with 2 flavors — a PINNED installation and PRERELEASE (local build mode). A production setup was added later. Both turned out to be problematic for new users and adopters, leading to confusion and a sub-par user experience. The following problems were identified:

1. **Complex and error-prone local-setup.** The local-setup is a set of bash scripts, kustomize overlays and YAML with multiple dependencies and 2 distinct modes (PINNED and PRERELEASE), where PRERELEASE is the default. Because of this inherent complexity, when the install fails it is hard for the user to understand what happened and how to fix it. A script-based imperative installation is also not a preferred practice and doesn't translate to other environments like PRODUCTION, air-gapped or GitOps-based installations.
2. **Huge resource requirements.** Platform Mesh requires 12 GB of memory and 6 CPUs allocated to Docker. This exceeds the defaults of most workstations and laptops, and the install fails if the user doesn't manually increase the limits. The resource footprint is unfit for most developer machines.
3. **Poor documentation.** The 3 different installation paths (PRODUCTION, local PINNED, and local PRERELEASE) were confusing and badly communicated. Users often started the wrong mode and tried to make it work on a topology where it was not intended. Issues on the website with wrong versions, the requirement to check out Git repositories before install, and duplication of docs across the website and multiple repositories all contributed to the bad user experience.
4. **No declarative GitOps installation.** A gap between user expectations and Platform Mesh documentation — this was simply missing.
5. **Oversized software stack.** The prerequisites list was large — KRO, OCM, Flux, openssl, mkcert, specific versions of jq, yq, and base64. If anything was missing or at the wrong version, installation often failed. The purpose of each tool was not clearly communicated, so users made changes in the wrong place (editing Flux HelmReleases instead of the PM profile), which made things worse. The larger and more complex the list of technologies, the heavier the mental burden and the worse the experience.

The goal is a Helm-first installation that uses only standard Kubernetes primitives, ships production-safe chart defaults, and lets GitOps controllers manage the lifecycle.

## Decision Drivers

- **Reduced resource footprint**: must run on most developer laptops without changes
- **A single installation path**: provide only one installation path, which can be tailored for local, production, air-gapped via configuration only
- **No image building during local install**: use published components for the install
- **Declarative-first**: every part of the installation must be expressible as a Kubernetes resource so GitOps controllers can manage and reconcile it.
- **Separation of production and development values**: the published chart ships production-safe defaults; environment-specific overrides live in files outside the chart.
- **Minimal prerequisites**: External dependencies are moved out of the PlatformMesh installation and listed clearly as requirements: OCM, FLUX, cert-manager, Ingress, Gateway-API, cloudnative-pg, keycloak-operator; Remove as much as possible dependencies: KRO
- **GatewayClass agnosticism**: the chart should not hard-code Traefik; it should accept a configurable GatewayClass name so users can bring their own ingress controller.
- **Global configuration via `PlatformMesh` resource**: settings that appear in multiple Helm charts (e.g. `userIdClaim`, IdP configuration) should be set once in the `PlatformMesh` resource and propagated by the operator, avoiding partial-configuration drift.
- **Improved user documentation**: website, repositories, readme's should all be consistent and straightforward. User should be able to follow obvious installation instructions without alternatives, duplicates and unclear installation modes.
- **No git checkout**: Users shouldn't have to git clone/checkout resources in order to install Platform Mesh.

## Considered Options

### A. Improved status quo — bootstrap scripts, production guide

Iterate on the existing shell scripts and production guide. Teams that need GitOps compatibility must create it by themselves.

- Good, because all most scenarios are covered: local pinned, local prerelease, production.
- Good, because of existing platform-mesh-operator already does 90% of the installation decleratively in a GitOps friendly way.
- Good, because it is fully automated.
- Bad, because of heavy installation scope - PM installs external dependencies which are not core PM.
- Bad, because users run into many issues and have bad experience.
- Bad, because the correct path is hard to identify and follow.
- Bad, because of too unrealistic local requirements for resources and dependency tooling.
- Bad, because host-tool differences (base64, jq) cause silent failures on developer machines.
- Bad, because if error occurs, debugging is hard for a newbee without experience with the whole stack.

### B. New declarative installation based on the platform-mesh-operator

Provide a single installation path which is declerative and uses the `platform-mesh-operator` to fully bootstrap a working environment. It can target different topologies via configuration. Removes external dependencies outside the PM installation path and clearly lists them as prerequisites. Keeps OCM and existing components structure. Charts defaults target production, while local needs configuration override. Further, a stripped down CORE PM installation can be enabled when modularization RFC is implemented and non-core functionalities can be turned off.

- Good, because a `helm install` command produces a complete, reconciling installation without scripts.
- Good, because the rendered Kubernetes resources are managed by Flux or any other GitOps controller.
- Good, because we keep OCM benefits like the chart can be pinned to an exact Platform Mesh OCM component version, making installations reproducible and air-gap-capable.
- Good, because it removes confusion regarding appropriate installation path.
- Good, because it reuses the `platform-mesh-operator` to bootstrap the environment which keeps the existing structure in place and features like operator templating.
- Good, because it enables the removal of many dependencies like KRO, jq, yq, mkcerts, openssl.
- Bad, because user might still get confused by unfamiliar technologies - OCM.
- Bad, because the user stills needs to understand how the installation process works and where the user-facing configuration toggles live.
- Bad, because useful features like local-setup and PRERELEASE modes are removed.
- Bad, because it is less automated compared to script-heavy installation.

### C. Flat Flux `HelmRelease` manifests applied directly

Ship Platform Mesh as a flat set of per-component Flux `HelmRelease` and `OCIRepository` manifests that are applied directly with `kubectl apply`, deliberately bypassing the convenience layers — no OCM resolution at runtime, no KRO, and no Platform Mesh Operator templating. A small Go installer generates the per-cluster manifests (prompting for base domain, front-proxy and Traefik service IPs) and can mirror the Platform Mesh OCM component — all sub-components, charts and images — into another OCI registry for air-gapped environments. Flux reconciles the ~24 `HelmReleases` into pods. This is the approach of the [`platform-mesh-from-scratch`](https://github.com/xrstf/platform-mesh-from-scratch) spike (validated against Platform Mesh 0.5.2 on a Gardener shoot).

- Good, because it is the absolute minimum needed to run Platform Mesh — every component is a plain `HelmRelease`, so there is nothing between the user and Flux to reason about.
- Good, because the Go installer handles OCM mirroring and generates matching Helm values, making air-gapped installs reproducible without hidden steps.
- Good, because the installation is fully declarative and GitOps-native once the manifests are generated.
- Bad, because stripping OCM/KRO/PMO templating means the per-cluster values (domain, service IPs, host aliases) must be substituted into every manifest, and the generator becomes a parallel source of truth to the published chart.
- Bad, because there is no composition engine ordering the `HelmReleases`; the rollout relies on Flux retries and raised concurrency, and several steps (kcp webhook Secret, `platform-mesh-profile.yaml`) are still manual in the spike.
- Bad, because it discards the Platform Mesh Operator's bootstrapping role rather than fixing it — the operator already performs 90% of the declarative install, so this duplicates effort instead of building on it.
- Bad, removing OCM controller loses some features like proof of origin, packaging, versioning.

## Decision Outcome

Chosen option: **B — New declarative installation based on the platform-mesh-operator**, because it satisfies the declarative and GitOps-compatibility requirements while keeping the initial installation experience simple, and because it builds on the Platform Mesh Operator's existing bootstrapping role rather than discarding it. Option C proves that a bare-manifest install is possible, but strips the convenience layers (OCM at runtime, PMO templating) that make the install reproducible and air-gap-friendly without a parallel generator; its value is as a reference for the minimal runtime dependencies, not as the shipped install path.

Concretely:

1. **`installation` stanza in the Platform Mesh Operator chart** (opt-in, default `false`). When enabled, the chart renders:
   - An OCM `Repository` pointing to `ghcr.io/platform-mesh`.
   - An OCM `Component` referencing the pinned `installation.version` and verifying the component signature.
   - A `PlatformMesh` resource wired to the OCM component.
   - The OCM signing public certificate (Secret), unless `installation.installSigningCertificate=false`.
   - The chart fails to render if `installation.baseDomain` is not set.

2. **Production-safe chart defaults only.** Values specific to local Kind clusters (host aliases, hard-coded Traefik ClusterIP, localhost port overrides) are removed from the chart. A local-development values file outside the published chart carries those overrides for developer and CI use.

3. **`default-profile.yaml` remains in `files/` for service wiring**, but must not contain any localhost or Kind-specific values. It describes how Platform Mesh services connect to each other; environment-specific overrides are applied on top.

4. **Local Kind TLS is handled by a separate, explicitly local chart** (`platform-mesh-kind-certificates`). It creates a self-signed CA and leaf certificate once cert-manager is ready, and must not be used in production. The two-step sequence (install operator → install certificates and restart operator) is kept for Kind only, until the operator can reload its Go trust store without a restart.

5. **Flux and the OCM Kubernetes controller are declared prerequisites.** The chart does not install them. An optional prerequisites chart is provided for users who want a reference starting point; production teams are expected to satisfy prerequisites through their own tooling.

6. **KRO is removed from the installation flow.** The Platform Mesh Operator renders `HelmRelease` resources directly from OCM components. KRO's template role is absorbed into the operator. The effective chain becomes: `PMO → OCM → HelmReleases → Flux → Pods`. Using KRO by end user to enable fully encapsulated installation based on OCM source is still easily achievable.

7. **Global configuration via `PlatformMesh` resource.** Settings that currently appear in multiple Helm charts (`userIdClaim`, IdP endpoint, SMTP) are promoted to fields on the `PlatformMesh` resource. The operator propagates these values to the relevant Helm releases, so operators set them in one place.

### Consequences

- Good, because a complete Platform Mesh installation for Kind requires two `helm install` commands and no scripts.
- Good, because the same chart, with different values, is used for local development, demo, and production; no separate code paths exist.
- Good, because Flux or ArgoCD can manage the lifecycle by supplying the `PlatformMesh` resource directly (`installation.enabled=false`) or by wrapping the chart in a `HelmRelease` (`installation.enabled=true`).
- Good, because the OCM component version is an explicit chart value, making rollbacks and upgrades fully declarative.
- Good, because removing KRO eliminates one reconciliation layer and simplifies the mental model to `PMO → OCM → HelmReleases → Flux → Pods`.
- Good, because global configuration in the `PlatformMesh` resource prevents the partial-configuration drift that has caused production incidents.
- Bad, because the two-step Kind installation (operator, then certificates and restart) is a deliberate sequencing dependency that cannot yet be eliminated without an operator change.
- Bad, because Traefik is still referenced in the current profile; making GatewayClass fully configurable requires follow-up work in the chart and operator.
- Bad, because of technology stack complexity is still hard to understand by all users.
- During the transition, `installation.enabled=false` remains the default to avoid breaking deployments that supply `PlatformMesh` resources externally.

## Open Questions

- **Production profile**: a production-appropriate profile (no Kind-specific values, no `hostAliases`, agnostic GatewayClass) needs to be defined and tested on an internet-facing cluster before the installation flow is recommended beyond Kind.
- **Traefik coupling**: the current profile has hard references to Traefik (GatewayClass name, ClusterIP). Making GatewayClass configurable is deferred to a follow-up PR.
- **Two-step Kind sequence**: the operator restart after certificate creation is a Go trust-store limitation. Eliminating it requires an operator change; deferred.
- **Prerequisites chart scope**: the optional prerequisites chart (Flux, OCM controller, Traefik, cert-manager) needs a defined scope. The question of whether it ships as part of the Platform Mesh component or as a separate reference artifact is open.
- **KRO migration**: operator logic to replace KRO's template rendering needs to be designed and implemented before KRO is removed. This ADR records the intent; the implementation is a separate work item.

## References

- [PR #2: script-less install (akafazov/platform-mesh-helm-charts)](https://github.com/akafazov/platform-mesh-helm-charts/pull/2) — POC implementation that this ADR documents.
- [`platform-mesh-from-scratch`](https://github.com/xrstf/platform-mesh-from-scratch) — the bare-manifest "from scratch" spike documented as option C.
- [ADR 001: SBOM Generation and OCM Component Restructuring](001-sbom-generation-and-ocm-component-restructuring.md) — the three-component OCM model that the installation flow depends on.
- [ADR 010: Consolidate Helm Charts and OCM into a Single Repository](010-consolidate-helm-charts-and-ocm.md) — the chart repository this installation flow is part of.
- [Platform Mesh Zulip: local-setup thread](https://linuxfoundation.zulipchat.com/#narrow/channel/532985-neonephos-platform-mesh-discussion/topic/local-setup/with/628570578) — extended community discussion that shaped this decision.
