# Repo split plan: core, control plane, actor, manufacturing services

Status: in progress. Stage 1 is on branch `core-independent-of-control-plane`.

## Decisions

| Question | Decision |
|---|---|
| Repos | **core** (versioned Python library), **control-plane**, **actor**, **manufacturing-services** |
| Domain code | Lives only in the **actor** repo. Never in core, never in the control plane. |
| Does the control plane run actors? | **No.** agentos keeps registry, scheduling, world state, auth and routing, and always forwards actor work to an Actor Runtime. |
| Domain HTTP APIs (commerce, orders, fulfillment, payment webhooks) | A separate **domain API service** in the actor repo |
| Manufacturing microservices | Their own **manufacturing-services** repo |
| Git history | Kept per repo (git filter-repo) |
| Hosting | Private repos under `monkeypatched-stack`; names confirmed before creation |

## Dependency rules (each enforced by a conformance check)

```
control-plane ──► core ◄── actor (domains, domain API service, device adapters)
manufacturing-services ──► core
```

- core imports neither the control plane nor domain code (`core_imports_control_plane` exists; `core_imports_domain` is added in stage 2)
- the control plane imports no domain code
- the actor never imports the control plane; it talks to it over HTTP via `actor_contract`

## Classification

**Core**: domain-agnostic runtime, shared by everything:
- `src/monkey_brain/kernel/` cognition loop, planning, pipeline, governance and security, society/planetary/world, scheduler and lifecycle, persistence, timelines, geography
- generic coordination: `kernel/edge/negotiation.py`, `society/negotiation_planning.py`, `pipeline/negotiation_store.py`, approvals, delegation
- generic edge infrastructure: sync, local store, offline safety, `actor_pod_client`, delegation cache
- `src/monkey_brain/{runtime,persistence,actor_contract,common,oql,routing,dashboard}` (minus the `routes.py` HTTP modules)
- `services/common` plus the shared auth primitives moved into it (tokens, revocation, rbac, agent_tokens, nats_store)
- `packages/*` libraries, `sdk/`

**Actor**: domain code and everything that runs only on an actor:
- `src/monkey_brain/actor_runtime.py`, `scripts/edge_actor.sh`, `start_actor.sh`, actor Docker images and manifests
- `src/monkey_brain/kernel/domains/*` (grocery, commerce, logistics, finance, payments, Razorpay, recall, robot, supply chain, support, vertical router)
- domain-specific code now inside the core, extracted behind extension points: grocery matching in `planning/context_engine.py`, `pipeline/payment_store.py`, `integrations/commerce.py`, `society/commerce_network.py`, domain intent predicates (`ecommerce_order`, `warehouse_shipping`, warehouse/work-order intent examples)
- device and modality adapters: `kernel/edge/{px4_ros_adapter,nav2_*,ros_*,drone_state,camera_state,livekit_*,video_*,voice_*}`
- `domains/` packages: manufacturing agents and knowledge (not services), software_engineering, domain manifests
- **domain API service**: `api/routes/{commerce,orders,fulfillment,payments}`, plus the domain parts of `actors.py` and `planet.py`

**Control plane**:
- `src/monkey_brain/api/` (without the domain routes) and the `routes.py` HTTP modules
- `services/agentos`, `services/auth` (service code; its shared helpers moved to core)
- `apps/workspace`, `living-world-explorer`, `kong/`, `opa/`, control-plane Docker images and k8s manifests, Helm charts (`somatic/`), `scripts/runpod.sh`, `scripts/stack.sh`

**Manufacturing services**: `domains/manufacturing/knowledge/services/*` except `common`, `agentos`, `auth`. Three platform modules (`services/auth` ×2, `services/common/agent_auth.py`) currently import `pm`, `module_control` and `file`; those imports get cut.

**Not migrated** (stays in the original repo as history unless asked otherwise): `.local/` (a committed MongoDB data directory), `generated/`, `tmp/`, `output/`, and the loose top-level reports.

## Stages

1. **Core stops importing the control plane.** Done on the branch: bootstrap moved to `kernel/bootstrap.py`, payments holder moved out of the HTTP route, `kernel/host_state.py` replaces kernel imports of `api.main`, shared auth helpers moved to `services/common`, `core_imports_control_plane` check added.
2. **Core stops importing domain code.** Add a domain plugin interface to the core (capabilities, agents, planning context providers, actor-registration hooks, execution engine), have the actor register its domains at startup, move domain modules to the actor side, and add a `core_imports_domain` check.
3. **Control plane never runs actors.** Remove agentos's local actor execution; it always forwards to an Actor Runtime. `runpod.sh` always starts a separate actor.
4. **Domain API service.** New FastAPI service on the actor side for the domain routes; Kong routes them there.
5. **Manufacturing services.** Cut the three platform → manufacturing imports and package the services on their own.
6. **Split** into four repos with history; core published as a tagged package that the others pin.
