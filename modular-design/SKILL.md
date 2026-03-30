Apply the modular design guidelines from `modular-design.md` to review or guide the current task.
TRIGGER when: adding a new domain, new cross-domain operation, new service interface, moving logic between layers, or any architectural decision about where code belongs.
DO NOT TRIGGER when: implementing within an existing domain with no structural changes.

---

# Modular Design Guidelines

These guidelines define the architectural approach for application design. Apply them when creating `architecture.md` or making technology and structural decisions for a project.

## Core Principle: Domain-Driven Modularity

Organize the application into **bounded domains** (DDD bounded contexts) that map to distinct business areas. Each domain encapsulates its own logic, data, and language. The goal is to **limit cognitive load** — a developer working in one domain should not need to understand the internals of another.

## Domain Rules

1. **Clean interfaces:** Each domain exposes a public API (interface/facade) that other domains and the application layer consume. Internal implementation details (repositories, entities, value objects, internal services) are not visible outside the domain.

2. **No cross-domain database access:** A domain owns its data. Other domains must go through the domain's public interface — never query another domain's tables directly. This applies even if the domains share a physical database.

3. **Business-aligned naming:** Unless a domain is simple CRUD, its public methods and value objects should reflect **actual business operations**, not technical operations. Use the language of the business problem (e.g., `reassignPerson(personId, targetTeamId, effectiveDate)` rather than `updatePersonTeamMapping(dto)`).

4. **Simple CRUD is fine:** Not every domain needs rich modeling. If a domain is genuinely just managing reference data (e.g., roles with a name and color), a thin CRUD layer is appropriate. Don't force complexity where there isn't any.

5. **Application layer as glue:** Cross-domain coordination happens in the **application layer** (use cases / application services), not inside domains. If an operation spans two domains, the application service calls each domain's interface and orchestrates the flow.

6. **Plugin architecture where appropriate:** For features that are likely to have multiple implementations, vary per deployment, or benefit from extensibility (e.g., authentication providers, export formats, notification channels), prefer a **plugin/adapter pattern** — define an interface in the domain or application layer and provide implementations as pluggable modules.

7. **Prefer graceful degradation over cross-domain coupling:** When a domain references data owned by another domain, consider what happens when that data disappears, changes, or becomes inconsistent. Instead of enforcing strict referential integrity across domain boundaries (which creates tight coupling, cascading operations, and cross-domain transactions), design domains to **tolerate stale or missing references gracefully**.

   Ask: "What if this domain only has an ID and the referenced entity is gone?" Often, the answer reveals a useful business concept rather than an error state. Examples:
   - A task assigned to a deleted user → an "unassigned task" (actionable, not broken)
   - An order referencing a discontinued product → a fulfilled order with historical line items (valid record)
   - A booking for a removed resource → a "ghost booking" signaling an unfilled need

   This approach yields concrete benefits:
   - **Domains stay independent.** No domain needs to notify, cascade to, or block on another domain's operations. Each domain can be developed, deployed, and reasoned about in isolation.
   - **Cross-domain operations vanish.** Instead of "when X is deleted, find and update all references in domains A, B, C," the referencing domain simply handles the missing reference at read time. The application layer assembles the full picture.
   - **External integration becomes safe.** If data is imported from external systems (HR, CRM, ERP), those systems can freely create or remove records without risking integrity violations in downstream domains.
   - **Conflict detection stays local.** Each domain validates only its own consistency. No need for cross-domain validation logic in the application layer.

   This is not eventual consistency in the distributed systems sense — it's a deliberate **design choice to relax referential assumptions** at domain boundaries, trading strict cross-domain consistency for independence and simplicity. Apply it when the "degraded" state is meaningful to the business, not when data corruption would go unnoticed.

## Practical Guidance

- **Start with domains, not layers.** Identify the bounded contexts first, then decide on the internal structure of each.
- **Shared kernel is small.** Cross-domain shared types (if any) should be minimal — primitives, IDs, and simple value objects at most.
- **Events for loose coupling.** When domains need to react to things happening in other domains, prefer **domain events** over direct calls where it reduces coupling without adding unnecessary complexity.
- **Don't over-engineer boundaries.** If two concepts are tightly coupled and always change together, they likely belong in the same domain. Splitting them would add integration overhead with no cognitive load benefit.
- **Challenge cascading requirements.** When you find yourself writing "when X happens in domain A, also do Y in domain B," stop and ask whether domain B truly needs to react, or whether it can simply tolerate the changed state. Cascading operations are the primary driver of cross-domain complexity. Every cascade you eliminate is a coupling you don't have to maintain.
- **Infrastructure is separate.** Database adapters, HTTP controllers, message brokers — these live outside the domain. The domain defines interfaces (ports); infrastructure provides implementations (adapters).
