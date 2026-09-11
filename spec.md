# cadastre

## Hierarchical Namespace and Resource Allocation Registry

**Status:** Draft v0.1  
**Implementation language:** Rust  
**Primary objective:** Implement a domain-independent registry for hierarchical resource allocation, with initial support for IP address/prefix allocation and Bundle Protocol IPN allocation.

---

# 1. Purpose

cadastre is a registry for describing, allocating, assigning, and tracking resources that exist within hierarchical namespaces or allocation spaces.

The initial motivating use cases are:

- IPv4 prefix allocation
- IPv6 prefix allocation
- Bundle Protocol IPN allocator blocks
- Bundle Protocol IPN node-number allocation
- Bundle Protocol IPN service-number allocation
- Mission/service definitions that consume identifiers from multiple resource spaces
- GitOps-oriented declarative configuration and review

cadastre is intended to be more general than an IPAM system. IPAM is one application of the underlying resource-allocation model.

The implementation should therefore avoid embedding IP-specific assumptions into the core domain model.

---

# 2. Design Principles

The implementation MUST follow these principles:

1. **The domain model is independent of persistence.**
2. **Filesystem/Git-backed storage is a first-class use case.**
3. **The initial implementation MUST NOT require PostgreSQL or another external database.**
4. **Resource spaces define allocation and uniqueness boundaries.**
5. **Resource kinds describe what sort of resource is represented within a resource space.**
6. **Resources are allocated from resource spaces.**
7. **Entities and services are not themselves resource-space members merely because they consume resources.**
8. **Assignments connect resources to entities, services, or other domain objects.**
9. **Containment/allocation hierarchy is distinct from arbitrary relationships.**
10. **Cross-resource-space relationships are allowed, but resources MUST NOT implicitly contain resources from unrelated spaces.**
11. **Identifiers that are derived from other identifiers SHOULD NOT be modeled as independently allocated resources unless there is a compelling reason.**
12. **The core library should be usable without a CLI, filesystem, database, or network service.**
13. **The first implementation should favor explicit domain types and correctness over generalized framework abstractions.**

---

# 3. Terminology

## 3.1 Domain

A **Domain** is a semantic grouping of related resource spaces and domain objects.

Examples:

- IP
- Bundle Protocol
- Internet Routing
- Spacecraft Mission

A domain is primarily organizational and semantic.

A Domain is NOT necessarily an allocation or uniqueness boundary.

For example:

```text
IP
├── IPv4 Resource Space
└── IPv6 Resource Space
```

The fact that IPv4 and IPv6 are both in the IP domain does not mean they share an allocation space.

---

## 3.2 Resource Space

A **Resource Space** is the fundamental allocation and uniqueness boundary.

It answers:

> From what universe is this resource allocated, and according to what rules is it unique?

Examples:

```text
IPv4 address/prefix space
IPv6 address/prefix space
IPN node-number space
IPN service-number space
ASN space
```

Resources from different resource spaces MUST NOT collide merely because their values happen to have similar representations.

For example:

```text
IPv4: 10.0.0.1
IPv6: 10.0.0.1
```

are not conflicting resources because they belong to different spaces.

Likewise:

```text
IPN node number 10
IPN service number 10
```

are independent allocations.

A resource space MAY define:

- allocation semantics
- uniqueness rules
- containment rules
- value representation
- valid resource kinds
- allocation granularity
- policy
- well-known values
- validation rules

---

## 3.3 Resource Kind

A **Resource Kind** identifies the semantic type of a resource within a resource space.

Examples:

```text
IPv4Prefix
IPv4Address
IPv6Prefix
IPv6Address
IPNAllocatorBlock
IPNNodeNumber
IPNServiceNumber
ASN
```

Resource kinds do not themselves define uniqueness. The containing resource space does.

A resource space MAY support multiple resource kinds when those kinds share a common allocation universe.

The implementation MUST NOT assume a one-to-one relationship between resource spaces and resource kinds.

---

## 3.4 Resource

A **Resource** is an actual value allocated or registered within a resource space.

Examples:

```text
10.20.0.0/16
10.20.1.0/24

IPN allocator block 1000–1999

IPN node number 1007

IPN service number 10
```

A resource has at minimum:

```text
id
resource_space
resource_kind
value
state
metadata
```

Resources SHOULD have stable cadastre IDs independent of their serialized representation.

The externally meaningful value remains part of the resource.

---

## 3.5 Entity

An **Entity** is a thing to which resources may be assigned.

Examples:

```text
Spacecraft-A
Ground-Station-1
Mission-Control
Operator-X
Organization-Y
```

Entities are not resources.

---

## 3.6 Service

A **Service** is a domain object representing a logical service that may consume identifiers or endpoints from multiple resource spaces.

Examples:

```text
Telemetry
Command
File Transfer
Navigation
Mission Operations
```

A Service MAY have multiple assignments, potentially from unrelated resource spaces.

For example:

```text
Telemetry Service
├── IPN service number: 10
├── DTN demux: "telemetry"
├── UDP port: 4556
└── IPv4 endpoint: 10.20.1.10
```

This is intentional.

The Service is the semantic object; the identifiers are allocations assigned to it.

---

## 3.7 Assignment

An **Assignment** connects a resource to a domain object.

Examples:

```text
10.20.1.10 -> Telemetry Service
IPN service number 10 -> Telemetry Service
IPN node number 1007 -> Spacecraft-A
IPv4 prefix 10.20.0.0/16 -> Mission-X
```

Assignments SHOULD be modeled independently from resources.

A resource may exist before it is assigned.

An assignment SHOULD have:

```text
id
resource
subject
state
metadata
validity/lifecycle information
```

The subject may initially be an Entity or Service.

---

## 3.8 Delegation

A **Delegation** represents authority over a resource or allocation range being granted to another entity.

Delegation is distinct from ownership and assignment.

For example:

```text
Registry
  delegates IPv4 /16
      ↓
Mission
  allocates /24s
      ↓
Spacecraft
  receives /24
```

The initial implementation MAY represent delegation simply enough to preserve authority relationships, but MUST NOT conflate delegation with assignment.

---

## 3.9 Allocation

An **Allocation** represents consuming a portion of a resource space or parent resource.

Allocation is concerned with the resource hierarchy.

Assignment is concerned with attaching an allocated resource to a subject.

These are different operations.

Example:

```text
10.20.0.0/16
    allocated to Mission-A

10.20.1.0/24
    allocated from Mission-A's /16

10.20.1.10
    assigned to Telemetry Service
```

---

# 4. Core Model

The conceptual model is:

```text
Domain
  │
  ├── Resource Space
  │      │
  │      ├── Resource Kind
  │      │
  │      └── Resource
  │
  ├── Entity
  │
  └── Service
          │
          └── Assignment
                  │
                  └── Resource
```

A Resource Space is the allocation boundary.

A Resource can participate in a hierarchy when its resource space defines an allocation/containment relationship.

A Service or Entity may be related to resources across multiple resource spaces.

---

# 5. Resource Space Requirements

Every resource MUST belong to exactly one Resource Space.

Every Resource Space MUST have:

```text
id
name
domain
resource kinds
allocation semantics
```

A Resource Space MUST define enough information for the validator to determine whether two resources conflict.

For hierarchical spaces, it MUST additionally define whether one resource can contain another.

---

# 6. Hierarchical Allocation

Some resource spaces naturally support hierarchy.

Examples:

```text
IPv4
10.0.0.0/8
  └── 10.1.0.0/16
        └── 10.1.1.0/24
              └── 10.1.1.10

IPN node numbers
allocator block 1000–1999
  └── node number 1007
```

The hierarchy MUST be interpreted according to the resource space.

cadastre MUST NOT implement arbitrary parent-child relationships as the mechanism for determining resource containment.

For example:

```text
IPv4 prefix -> IPv6 prefix
```

is not a containment relationship.

If two resources belong to different resource spaces, they may be related explicitly, but they are not implicitly parent/child.

---

# 7. Cross-Space Relationships

Cross-space relationships are explicitly supported.

Examples:

```text
IPv4 address
    assigned-to
Telemetry Service

IPN service number
    assigned-to
Telemetry Service

DTN demux string
    assigned-to
Telemetry Service
```

This produces a graph of domain relationships without merging the underlying allocation spaces.

A resource MUST NOT gain containment semantics merely because it is related to another resource.

---

# 8. Initial Resource Spaces

The initial implementation MUST support the following.

## 8.1 IPv4

Resource space:

```text
ip/ipv4
```

Initial resource kind:

```text
prefix
```

The implementation SHOULD permit future support for:

```text
address
range
```

without requiring them now.

An IPv4 prefix:

```text
10.0.0.0/8
```

is a resource whose value has CIDR semantics.

The following are conflicts:

```text
10.0.0.0/8
10.1.0.0/16
```

if both are competing allocations in the same allocation hierarchy.

The following do not conflict:

```text
10.0.0.0/8
192.168.0.0/16
```

---

## 8.2 IPv6

Resource space:

```text
ip/ipv6
```

Initial resource kind:

```text
prefix
```

IPv6 prefixes MUST use canonical binary/network semantics for comparison.

String formatting MUST NOT be used to determine overlap.

---

## 8.3 IPN Node Number Space

Bundle Protocol IPN node allocation MUST distinguish allocator identity from node identity.

An FQNN is:

```text
allocator_id.node_id
```

The service number is NOT part of the FQNN.

The initial model SHOULD represent:

```text
IPN Node Number Space
├── Allocator Block
└── Node Number
```

An allocator block is a range of node numbers under the authority of an allocator.

Example:

```text
Allocator ID: 42
Node block: 1000–1999

Node:
  allocator_id = 42
  node_id = 1007

FQNN:
  42.1007
```

The FQNN SHOULD be modeled as a derived value from its allocator and node number rather than as an independently allocated resource.

---

## 8.4 IPN Service Number Space

The service number is independently allocated from an IPN service-number space.

Example:

```text
Service Number:
  10
```

A service number is NOT part of an FQNN.

A service endpoint conceptually combines:

```text
FQNN
+
Service Number
```

The exact representation of an endpoint MUST remain separate from the allocation of either component.

---

# 9. Services

Services are first-class domain objects.

A Service SHOULD contain:

```text
id
name
description
owner/entity
lifecycle state
metadata
assignments
```

Assignments may come from different resource spaces.

Example:

```yaml
service:
  id: telemetry
  name: Telemetry
  owner: spacecraft-sc1

  assignments:
    - resource: ipn-service-10
    - resource: dtn-demux-telemetry
    - resource: udp-port-4556
```

Not every identifier associated with a service must necessarily be allocated by cadastre.

The model MUST allow externally defined or informational identifiers.

---

# 10. Well-Known Values

Some resource spaces may define well-known values.

For example:

```text
IPN service number:
    telemetry -> 10
    command   -> 20
    file      -> 30
```

A well-known value is a policy associated with a resource space or allocation authority.

It is NOT inherently a property of the Service.

A Service may request a well-known identifier, after which the resource space's policy determines whether that identifier is available and valid.

The initial implementation may support well-known values as declarative metadata without implementing sophisticated policy selection.

---

# 11. Lifecycle

Resources, assignments, entities, and services SHOULD have lifecycle state.

Initial states:

```text
planned
reserved
allocated
assigned
deprecated
released
```

The exact state transition rules should be centralized in domain logic.

At minimum:

```text
planned -> reserved
reserved -> allocated
allocated -> assigned
assigned -> deprecated
allocated -> released
assigned -> released
```

The implementation MUST prevent invalid transitions.

A released resource may subsequently be reallocated according to resource-space policy.

Historical assignments MUST NOT be silently overwritten.

---

# 12. Ownership, Authority, and Assignment

These concepts MUST remain distinct.

### Authority

Who is allowed to allocate/delegate the resource.

### Ownership

Who is responsible for or owns the resource.

### Assignment

Which entity/service currently uses the resource.

Example:

```text
NASA
  authority over allocator block

Mission-A
  owner of node 1007

Telemetry Service
  assignment of service number 10
```

The implementation MUST NOT use a single `owner` field to represent all three concepts.

---

# 13. Conflict Detection

The validator MUST detect allocation conflicts.

For a hierarchical numeric or prefix-based resource space, conflicts include:

- identical resources
- overlapping ranges
- a resource allocated outside its parent's extent
- incompatible resource kinds
- conflicting active allocations
- duplicate assignments where uniqueness policy forbids them

Examples:

```text
10.0.0.0/16
10.0.1.0/24
```

may conflict if both are siblings competing for the same allocation extent.

Whether parent/child allocations are allowed simultaneously MUST be defined by the resource-space allocation policy.

The implementation MUST NOT hard-code a universal "overlap is always invalid" rule.

---

# 14. Validation

cadastre MUST provide validation independent of persistence.

Validation SHOULD include:

## Structural validation

- required IDs exist
- referenced objects exist
- resource space exists
- resource kind is valid for the resource space
- entity/service references resolve
- duplicate IDs are rejected

## Allocation validation

- resource values are syntactically valid
- resources belong to the correct space
- allocations do not violate containment
- conflicting allocations are detected
- resource state is consistent

## Assignment validation

- assignment references an existing resource
- assignment references an existing subject
- assignment lifecycle is valid
- uniqueness rules are respected

## Domain-specific validation

IPv4:

- valid IPv4 network
- valid prefix length
- canonical network address

IPv6:

- valid IPv6 network
- valid prefix length
- canonical network address

IPN:

- valid allocator ID
- valid node number
- valid allocator block
- node number belongs to its allocator block
- service number is distinct from node number
- FQNN is correctly derived

---

# 15. Filesystem Representation

Filesystem storage is a primary use case.

The filesystem representation MUST be:

- human-readable
- deterministic
- Git-friendly
- diff-friendly
- independently validatable
- suitable for code review

A proposed layout:

```text
cadastre/
├── cadastre.yaml
│
├── domains/
│   ├── ip.yaml
│   └── bp.yaml
│
├── spaces/
│   ├── ip/
│   │   ├── ipv4/
│   │   │   ├── space.yaml
│   │   │   └── resources/
│   │   └── ipv6/
│   │       ├── space.yaml
│   │       └── resources/
│   │
│   └── bp/
│       └── ipn/
│           ├── node-numbers/
│           │   ├── space.yaml
│           │   └── resources/
│           └── service-numbers/
│               ├── space.yaml
│               └── resources/
│
├── entities/
│   ├── spacecraft-sc1.yaml
│   └── ground-station-1.yaml
│
├── services/
│   ├── telemetry.yaml
│   └── command.yaml
│
└── assignments/
    ├── telemetry-ipn.yaml
    └── telemetry-ip.yaml
```

This layout is illustrative rather than an immutable API.

The filesystem store SHOULD allow reasonable organization without requiring one file per resource.

---

# 16. Serialization

YAML SHOULD be the initial human-editable serialization format.

However:

> The Rust domain model MUST NOT depend on YAML.

Serialization should occur at the storage boundary.

The implementation SHOULD use strongly typed serialization/deserialization rather than untyped maps wherever practical.

JSON support MAY be added later.

---

# 17. Example Resource

An IPv4 prefix might be represented approximately as:

```yaml
id: ipv4-mission-a
space: ip/ipv4
kind: prefix

value: 10.20.0.0/16

state: allocated

metadata:
  description: Mission A IPv4 allocation
```

---

# 18. Example Entity

```yaml
id: spacecraft-sc1
kind: spacecraft

name: Spacecraft 1

metadata:
  mission: example-mission
```

The `kind` field here is an Entity classification, not a Resource Kind.

---

# 19. Example Service

```yaml
id: telemetry
name: Telemetry

owner: spacecraft-sc1

metadata:
  description: Spacecraft telemetry service
```

Assignments are maintained separately or embedded where appropriate, but the implementation MUST preserve assignments as first-class domain objects.

---

# 20. Example IPN Allocation

```yaml
id: ipn-node-1007

space: bp/ipn/node-numbers
kind: node-number

value:
  allocator_id: 42
  node_id: 1007

state: allocated
```

The corresponding FQNN is derived:

```text
42.1007
```

It is not a separate allocation.

---

# 21. Example Service Assignment

```yaml
id: telemetry-ipn-service

resource: ipn-service-10
subject: service/telemetry

state: assigned
```

This expresses:

```text
IPN service number 10
        assigned to
Telemetry Service
```

It does not imply that service number 10 is part of the FQNN.

---

# 22. Rust Architecture

The implementation SHOULD initially be organized approximately as:

```text
src/
├── lib.rs
│
├── domain/
│   ├── mod.rs
│   ├── id.rs
│   ├── domain.rs
│   ├── space.rs
│   ├── kind.rs
│   ├── resource.rs
│   ├── entity.rs
│   ├── service.rs
│   ├── assignment.rs
│   ├── allocation.rs
│   ├── delegation.rs
│   ├── lifecycle.rs
│   └── validation.rs
│
├── spaces/
│   ├── mod.rs
│   ├── ipv4.rs
│   ├── ipv6.rs
│   └── ipn.rs
│
├── registry/
│   ├── mod.rs
│   └── registry.rs
│
├── storage/
│   ├── mod.rs
│   └── filesystem.rs
│
└── cli/
    ├── mod.rs
    └── ...
```

The exact module structure may be changed if the implementation benefits from a better organization.

Do not create dozens of generic traits merely to anticipate hypothetical backends.

---

# 23. Domain Types

The implementation SHOULD use explicit newtypes for important identifiers.

For example:

```rust
struct DomainId(...);
struct ResourceSpaceId(...);
struct ResourceKindId(...);
struct ResourceId(...);
struct EntityId(...);
struct ServiceId(...);
struct AssignmentId(...);
```

Avoid passing arbitrary `String` values throughout the domain.

IDs should be stable and serialization-friendly.

UUIDs, ULIDs, or another stable identifier scheme may be selected by the implementation, provided the choice is documented.

---

# 24. Resource Values

Resource values SHOULD be represented using typed domain values rather than strings.

For example:

```rust
enum ResourceValue {
    Ipv4Prefix(Ipv4Net),
    Ipv6Prefix(Ipv6Net),
    IpnAllocatorBlock(IpnAllocatorBlock),
    IpnNodeNumber(IpnNodeNumber),
    IpnServiceNumber(IpnServiceNumber),
}
```

The exact enum structure may evolve.

The implementation MUST preserve enough type information to prevent accidental comparison between unrelated resource spaces.

---

# 25. Allocation API

The core registry should expose operations conceptually equivalent to:

```text
create_resource
reserve_resource
allocate_resource
release_resource
assign_resource
unassign_resource
delegate_resource
validate
```

Allocation SHOULD be performed through domain logic rather than by directly mutating stored objects.

Example:

```rust
registry.allocate(space, request)
```

The implementation should return a domain result describing the allocation.

---

# 26. Persistence Boundary

The domain layer MUST NOT depend on a specific persistence technology.

The initial implementation should provide:

```text
in-memory registry
filesystem registry
```

A future database implementation may provide the same conceptual operations.

Do not design a generic ORM abstraction.

A storage interface should expose domain-relevant operations rather than SQL-shaped CRUD.

For example, prefer:

```rust
load_registry()
save_registry()
```

or domain-specific repository operations over:

```rust
insert_row()
update_row()
select_where()
```

---

# 27. GitOps

Git itself is not the storage abstraction.

Instead:

```text
cadastre filesystem store
        +
       Git
        =
GitOps workflow
```

The repository should support workflows such as:

```bash
cadastre validate
cadastre diff
cadastre plan
cadastre apply
```

The first implementation MUST prioritize:

```bash
cadastre validate
```

and basic inspection/query commands.

`diff`, `plan`, and `apply` may initially be limited in scope.

---

# 28. CLI

The initial CLI should provide at least:

```text
cadastre validate
cadastre list
cadastre show
cadastre inspect
```

Recommended future commands:

```text
cadastre allocate
cadastre reserve
cadastre release
cadastre assign
cadastre unassign
cadastre delegate
cadastre diff
cadastre plan
cadastre apply
```

Commands should operate against a specified registry directory.

Example:

```bash
cadastre validate ./cadastre
```

The CLI MUST return a non-zero exit code when validation fails.

Validation errors should identify:

- object
- field/value when relevant
- violated invariant
- useful remediation information

---

# 29. Querying

The registry should support basic queries such as:

```text
list all IPv4 allocations
find resource by ID
find resources containing an address
find assignments for a service
find services using a resource
find all resources belonging to an entity
```

The query API should be domain-oriented.

Do not optimize prematurely for a database query language.

---

# 30. Audit and Events

The design SHOULD support domain events.

Examples:

```text
ResourceCreated
ResourceReserved
ResourceAllocated
ResourceReleased
ResourceAssigned
ResourceUnassigned
ResourceDelegated
```

Events are useful for:

- audit history
- GitOps reconciliation
- future event streaming
- synchronization
- debugging

The first implementation does NOT need to be event-sourced.

The current state may remain authoritative.

---

# 31. Error Handling

Errors SHOULD distinguish at least:

```text
NotFound
AlreadyExists
InvalidValue
InvalidStateTransition
AllocationConflict
ContainmentViolation
InvalidAssignment
InvalidReference
ValidationError
SerializationError
StorageError
```

Domain errors should not expose filesystem or database implementation details.

---

# 32. Testing

The implementation MUST have unit tests for:

## IPv4

- canonicalization
- prefix overlap
- containment
- allocation conflicts
- adjacent non-overlapping prefixes

## IPv6

- canonicalization
- prefix overlap
- containment
- allocation conflicts

## IPN

- allocator block containment
- node-number allocation
- FQNN derivation
- service-number allocation
- distinction between node and service numbers

## Assignments

- assigning existing resources
- invalid references
- duplicate assignments
- lifecycle behavior

## Cross-space relationships

Tests MUST demonstrate that:

```text
IPv4 10.0.0.1
IPv6 equivalent textual representation
IPN node number 10
IPN service number 10
```

are independent resources because they belong to different resource spaces.

## Filesystem

Tests MUST cover:

- loading valid registries
- rejecting malformed files
- validation failures
- deterministic serialization
- round-trip serialization

---

# 33. Determinism

The filesystem store MUST produce deterministic output.

Given the same domain state, serialization should produce equivalent output regardless of:

- hash-map iteration order
- process execution
- object insertion order

This is important for Git diffs.

---

# 34. Concurrency

The initial implementation does not need distributed locking.

An in-memory registry may use ordinary Rust ownership/borrowing semantics.

Filesystem writes should avoid corrupting an existing registry.

Atomic replacement of generated files SHOULD be used where practical.

Distributed concurrent allocation is explicitly out of scope for v0.1.

---

# 35. Security

Authentication and authorization are out of scope for the first implementation.

However, the domain model MUST distinguish authority from ownership and assignment so that authorization can be added later without redesigning the model.

---

# 36. Non-Goals

The following are explicitly NOT required for the initial implementation:

- PostgreSQL
- MySQL
- REST API
- GraphQL
- Web UI
- Authentication
- Multi-user access control
- Distributed locking
- High availability
- Event sourcing
- Message queues
- Kubernetes integration
- Cloud-specific integrations
- Automatic DNS management
- Automatic routing configuration
- Full DTN protocol implementation
- Full Bundle Protocol implementation
- Network configuration
- SNMP
- DHCP
- BGP integration
- Automatic spacecraft configuration
- Cryptographic identity management

These may be future integrations.

---

# 37. Important Non-Goal: Generic Everything

The implementation MUST NOT attempt to make every conceivable allocation problem fit into a giant generic trait hierarchy.

For example, do not prematurely introduce abstractions such as:

```rust
trait Allocatable<T, P, C, V, A, ...>
```

simply because future resource types might exist.

The abstraction should emerge from concrete requirements.

The initial concrete resource spaces are sufficient to determine the correct abstractions.

---

# 38. Initial Milestones

## Milestone 1 — Core Domain

Implement:

```text
IDs
Domain
ResourceSpace
ResourceKind
Resource
Entity
Service
Assignment
Lifecycle
Validation
```

No persistence.

Tests should establish the basic invariants.

---

## Milestone 2 — IP Resource Spaces

Implement:

```text
IPv4 prefix
IPv6 prefix
```

including:

- parsing
- canonicalization
- containment
- overlap
- allocation conflict detection

Use a mature Rust IP/network library rather than implementing IP arithmetic from scratch.

---

## Milestone 3 — IPN

Implement:

```text
IPN allocator blocks
IPN node numbers
IPN service numbers
FQNN derivation
```

Tests must explicitly demonstrate:

```text
FQNN = allocator_id + node_id
```

and:

```text
service number != node number
```

---

## Milestone 4 — Services

Implement:

```text
Service
Service -> Assignment -> Resource
```

Allow one Service to have assignments from multiple resource spaces.

Demonstrate with:

```text
Telemetry
├── IPN service number
└── another identifier
```

---

## Milestone 5 — Filesystem Store

Implement loading and writing the declarative filesystem representation.

Support:

```bash
cadastre validate ./cadastre
cadastre list ./cadastre
cadastre show ./cadastre <id>
```

Round-trip tests are required.

---

## Milestone 6 — Allocation Operations

Implement CLI/API operations for:

```text
reserve
allocate
release
assign
unassign
```

All operations MUST pass domain validation before persistence.

---

## Milestone 7 — GitOps Workflow

Implement:

```bash
cadastre diff
cadastre plan
cadastre apply
```

where appropriate.

The filesystem remains the authoritative representation.

Git remains an external version-control mechanism.

---

# 39. Example Initial Registry

A minimal registry should be capable of representing something like:

```text
Mission: Example Mission

Entities:
  spacecraft-sc1

Resource spaces:
  IP / IPv4
  IP / IPv6
  BP / IPN node numbers
  BP / IPN service numbers

Resources:
  IPv4 10.20.0.0/16
  IPN allocator block 42:1000-1999
  IPN node number 42:1007
  IPN service number 10

Services:
  telemetry

Assignments:
  42:1007 -> spacecraft-sc1
  IPN service number 10 -> telemetry
  IPv4 10.20.0.0/16 -> spacecraft-sc1
```

The derived FQNN is:

```text
42.1007
```

The service endpoint relationship is conceptually:

```text
FQNN 42.1007
    +
service number 10
```

The service may additionally have a DTN demux identifier:

```text
telemetry
```

That demux identifier may eventually be modeled as another resource space or as a typed identifier allocation, but the first implementation does not need to settle this question prematurely.

---

# 40. Open Questions

The following questions are intentionally unresolved.

## 40.1 Are all identifier types Resources?

Some identifiers clearly belong in resource spaces:

```text
IPv4 address
ASN
IPN service number
```

Others may be better represented as identifiers associated with domain objects.

Examples:

```text
DTN demux string
DNS name
URI
TCP/UDP port
```

The implementation should not force these into ResourceSpace until there is a demonstrated allocation/uniqueness requirement.

---

## 40.2 Is a Prefix a Resource or an Allocation Container?

IPv4 and IPv6 prefixes have dual semantics:

```text
they are resources themselves
and
they contain smaller allocatable resources
```

The model should support this without requiring separate artificial "container" objects.

---

## 40.3 Are IP Addresses and Prefixes One Resource Space?

A possible future model is:

```text
IPv4 Space
├── Prefix
└── Address
```

because both are coordinates in the same address universe.

Another possibility is separate spaces with an explicit containment relationship.

The implementation should avoid making either choice impossible.

---

## 40.4 What Exactly Is an IPN Allocator?

The BP/IPN model may eventually need a first-class Allocator object rather than treating allocator ID as merely a numeric field.

This should be investigated before implementing complex delegation.

---

## 40.5 Are FQNNs Resources?

Current assumption:

> No. FQNNs are derived identifiers from an allocator ID and node ID.

This should remain the default unless a use case demonstrates that FQNNs need independent lifecycle/allocation semantics.

---

## 40.6 How Should Well-Known Values Work?

Possible model:

```text
Resource Space
  well-known assignments
```

versus:

```text
Resource Space
  allocation policy
      -> well-known values
```

The first implementation can use declarative metadata and defer sophisticated policy evaluation.

---

## 40.7 Should Services Be Hierarchical?

Mission planning may naturally produce:

```text
Mission
  Spacecraft
    Subsystem
      Service
        Endpoint
```

This is likely useful, but arbitrary hierarchical modeling of domain objects should not be implemented until concrete use cases require it.

---

## 40.8 What Is the Relationship Between Entity and Service?

A Service may be hosted by an Entity:

```text
Telemetry
  hosted-by
Spacecraft-1
```

and may communicate with other Entities.

This suggests a general relationship model may eventually be useful:

```text
Relationship {
    subject
    predicate
    object
}
```

but a generic graph engine is not required for v0.1.

---

## 40.9 Should Domains Be Persistent Objects?

A Domain may ultimately be little more than metadata used to organize Resource Spaces.

The implementation should keep Domain lightweight and avoid giving it allocation semantics unless required.

---

# 41. Implementation Guidance for the Coding Agent

When beginning implementation:

1. Read this specification completely.
2. Do not begin by designing a database schema.
3. Do not begin by implementing an HTTP server.
4. Do not create a web UI.
5. Establish the domain model and invariants first.
6. Write tests before or alongside each domain capability.
7. Implement IPv4/IPv6 and IPN as concrete examples that exercise the generic model.
8. Prefer explicit types over premature generic traits.
9. Keep serialization at the boundary.
10. Implement the in-memory domain before filesystem persistence.
11. Make filesystem serialization deterministic.
12. Ensure validation can run without writing anything.
13. Keep cross-resource-space relationships explicit.
14. Do not model FQNN as containing a service number.
15. Treat Service as a first-class domain object.
16. Preserve the distinction between authority, ownership, allocation, and assignment.
17. When encountering an ambiguity not resolved by this specification, prefer the smallest implementation that preserves future flexibility.
18. Document significant deviations from this specification in code comments or design notes.

---

# 42. Definition of Done for v0.1

The initial implementation is complete when an agent can create a filesystem registry containing:

```text
an IPv4 allocation
an IPv6 allocation
an IPN allocator block
an IPN node number
an IPN service number
an Entity
a Service
assignments connecting those resources to the Entity/Service
```

and:

```bash
cadastre validate ./cadastre
```

correctly detects:

- malformed resources
- invalid references
- duplicate IDs
- invalid IP networks
- overlapping IP allocations
- invalid IPN node allocations
- invalid allocator-block containment
- invalid lifecycle transitions
- invalid assignments

while successfully validating a correct registry.

The implementation must also be able to load the registry, expose its domain objects through the Rust API, and serialize it back deterministically.

The resulting code should make it straightforward to add another resource space—such as ASN allocation—without changing the semantics of existing IP or IPN resource spaces.

---

# 43. Guiding Architectural Statement

The implementation should preserve the following conceptual model:

> **cadastre is a registry of resources allocated from distinct resource spaces and assigned to domain objects.**

Resource spaces establish uniqueness and allocation semantics.

Resources represent actual allocated values.

Entities and Services represent things that consume those resources.

Assignments connect the two.

Domains organize related concepts but are not themselves necessarily allocation boundaries.

Hierarchy describes containment where the underlying resource space supports it.

Relationships connect otherwise independent objects without collapsing their resource spaces.

This model should remain the foundation of the implementation.
