## Phase 1 — Discovery & Analysis (Weeks 1–3)

### Goal
Understand the full landscape of what exists before writing a single line of Java. You cannot migrate what you haven't mapped.

### Activities

#### 1.1 — Codebase Inventory
Walk the entire .NET solution and catalogue every WCF service. You are looking for:

- Every `.svc` file (each one becomes a Spring Boot microservice candidate)
- Every `Reference.cs` file (each one maps to a SAP WSDL)
- Every `web.config` binding (tells you security mode, timeouts, endpoint URLs)
- Shared libraries and utilities (these become shared Spring Boot modules or a common library)

**AI role here:** Point Copilot Chat at the entire solution with `@workspace` and ask it to produce a structured inventory. A prompt like *"List all WCF service contracts, their operations, and the SAP endpoints they call, formatted as JSON"* will save days of manual reading.

**Output:** A service inventory document like this:

```json
{
  "services": [
    {
      "name": "PaymentService",
      "svcFile": "PaymentService.svc.cs",
      "operations": ["ProcessPayment", "ReversePayment"],
      "sapWsdl": "PaymentPosting.wsdl",
      "binding": "basicHttpBinding",
      "security": "UsernameToken",
      "timeout": "60s",
      "sharedDependencies": ["ValidationHelper", "AuditLogger"]
    },
    {
      "name": "AccountService",
      "svcFile": "AccountService.svc.cs",
      "operations": ["GetAccountDetails", "UpdateAccount"],
      "sapWsdl": "AccountManagement.wsdl",
      "binding": "wsHttpBinding",
      "security": "Certificate",
      "timeout": "30s",
      "sharedDependencies": ["AuditLogger"]
    }
  ]
}
```

This JSON becomes the **migration manifest** — the input to every subsequent phase.

---

#### 1.2 — Dependency Graph
Map which services call each other, and which share common utilities. This drives your microservice decomposition decisions.

```
PaymentService ──────────────────► SAP PaymentPosting
      │
      └──► ValidationHelper (shared)
      └──► AuditLogger (shared)

AccountService ──────────────────► SAP AccountManagement
      │
      └──► AuditLogger (shared)

StatementService ────────────────► SAP StatementRetrieval
      │
      └──► AccountService (internal call)
      └──► AuditLogger (shared)
```

**Critical decision point:** Internal WCF-to-WCF calls need careful thought. In the migrated world, do they become REST calls between microservices, or do they get consolidated? This is an architecture decision no AI can make for you — it depends on your team's domain boundaries.

---

#### 1.3 — SAP WSDL Audit
Collect every WSDL your .NET app consumes from SAP. For each one, check:

- Is it still the current version on the SAP side?
- Does it use any deprecated SAP operations?
- What security scheme does it use (UsernameToken, certificate, none)?
- Are there any size or timeout constraints on the SAP side?

**Why this matters:** If a WSDL has changed since the .NET client was generated, your `Reference.cs` may be out of sync with what SAP actually serves. Regenerating Java stubs from the live WSDL is the right move — not copying the .NET types across.

**Output:** WSDL registry — a simple table mapping each WSDL to its SAP endpoint URL, version, and security scheme.

---

#### 1.4 — Risk Classification
Score each service for migration complexity. A simple three-tier model works well:

| Tier | Criteria | Example |
|---|---|---|
| Green | Simple CRUD, one SAP call, no shared state | `StatementService` |
| Amber | Multiple SAP calls, shared utilities, some business logic | `AccountService` |
| Red | Complex orchestration, multiple internal calls, financial calculations | `PaymentService` |

Migrate Green services first. They prove the framework works and build team confidence before tackling the complex ones.

---

#### Phase 1 Outputs

- Migration manifest JSON (input to all future phases)
- WSDL registry
- Dependency graph
- Risk classification per service
- Architecture decision record for inter-service call strategy

---
---

## Phase 2 — Pilot Migration (Weeks 4–9)

### Goal
Prove the full pipeline works end to end on 2–3 Green-tier services before committing to the full migration. Every template and pattern you establish here gets reused across all remaining services.

### Why Pilot First
The worst thing you can do is migrate 20 services in parallel only to discover your Helm chart template has a fundamental flaw, or your SAP WS-Security config doesn't work with one particular SAP endpoint. Fix these problems cheaply on two services, not expensively on twenty.

---

### Activities

#### 2.1 — Build the Common Foundation (Do Once, Reuse Forever)

Before migrating any service, build the shared scaffolding that every microservice will inherit.

**Common Spring Boot parent module:**
```
common-lib/
├── SapWebServiceConfig.java      ← WebServiceTemplate, Wss4j setup
├── SapSecurityInterceptor.java   ← reusable for all SAP clients
├── BasePaymentException.java     ← common exception hierarchy
├── AuditLogger.java              ← migrated from .NET shared util
└── CorrelationIdFilter.java      ← request tracing for Istio
```

**Common Helm library chart:**
```
helm-library/
├── Chart.yaml
└── templates/
    ├── _deployment.yaml      ← AKS deployment with Istio labels
    ├── _service.yaml         ← Kubernetes service
    ├── _hpa.yaml             ← horizontal pod autoscaler
    ├── _virtualservice.yaml  ← Istio VirtualService
    └── _destinationrule.yaml ← Istio DestinationRule
```

Every microservice's Helm chart then becomes just a thin values file:
```yaml
# payment-service/values.yaml
image:
  repository: yourregistry.azurecr.io/payment-service
  tag: 1.0.0

replicas: 2

sap:
  endpoint: https://sap-host/sap/bc/srt/rfc/sap/payment_posting

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 500m
```

This is the **most reusable investment** of the entire programme. Get this right in Phase 2 and every subsequent service deployment is near-zero effort.

---

#### 2.2 — Run the Code Generation Pipeline on Pilot Services

For each pilot service, execute the generation steps in order.

**Step 1 — Generate SAP stubs from WSDL**
```bash
# Run jaxb2-maven-plugin against the SAP WSDL
# This replaces svcutil.exe / Reference.cs entirely
mvn generate-sources

# Output: target/generated-sources/xjc/
# com/yourorg/payment/sap/generated/
#   PostPaymentRequest.java    ← was in Reference.cs
#   PostPaymentResponse.java
#   PaymentFault.java
#   ObjectFactory.java
```

**Step 2 — Use Copilot to translate business logic**

Open the `.svc.cs` file in your IDE and use Copilot agent mode with a structured prompt:

```
Translate this C# WCF service implementation to Java 21 Spring Boot.
Follow these rules:
- Use Java records for all DTOs
- Use @Service for the business logic class
- Use @RestController for the endpoint
- Map FaultException<T> to a custom RuntimeException subclass
- Use WebServiceTemplate for the SAP client
- Preserve all comments explaining business logic
- Flag any C# patterns you are unsure how to translate with a TODO comment
```

Copilot will produce an 80–90% complete translation. The remaining 10–20% will be the business logic nuances and SAP-specific edge cases that need engineer review.

**Step 3 — Engineer review pass**

This is non-negotiable. For every AI-generated service:
- Verify business logic is semantically equivalent, not just syntactically translated
- Check all SAP field mappings are correct
- Verify error handling covers the same fault scenarios as the .NET original
- Confirm no silent type conversion issues (C# `decimal` to Java `BigDecimal` is usually fine, but watch for rounding mode differences in financial calculations)

---

#### 2.3 — Build the Contract Test Suite

This is your regression safety net for the entire migration programme.

For each pilot service, generate a Spring Cloud Contract spec from the WSDL operations:

```groovy
// contracts/processPayment_success.groovy
Contract.make {
    description "Process valid payment - SAP returns document number"

    request {
        method POST()
        url "/api/v1/payments/process"
        headers { contentType applicationJson() }
        body(
            accountNumber: "GB12NWBK60161331926819",
            amount: 150.00,
            currency: "GBP",
            costCentre: "CC001"
        )
    }

    response {
        status OK()
        body(
            documentNumber: anyNonBlankString(),
            status: "SUCCESS",
            message: anyNonBlankString()
        )
    }
}
```

These contracts serve two purposes. First, they verify your new Spring Boot service behaves correctly. Second, they become the permanent regression suite — if a future change breaks a contract, you know immediately.

---

#### 2.4 — SAP Integration Testing

Test against a SAP sandbox or development client. This is the highest-risk area because SAP SOAP behaviour has quirks that no amount of unit testing will catch:

- SAP sometimes returns non-standard fault structures that diverge from the WSDL
- SAP's WS-Security implementation can be strict about timestamp tolerances
- Some SAP operations have mandatory header fields not declared in the WSDL (added via custom enhancers in the .NET code — check for `IClientMessageInspector` implementations in the .NET codebase, these are easy to miss)

**AI role:** Copilot cannot help you here. This requires a human engineer with access to the SAP sandbox, running real calls and comparing responses between the legacy .NET client and your new Spring Boot client.

---

#### Phase 2 Outputs

- Working common-lib module
- Working Helm library chart
- Working Harness pipeline template
- 2–3 fully migrated and tested Green-tier services in production
- Copilot instruction file refined based on what worked and what didn't
- Known SAP integration edge cases documented for all remaining services

---
---

## Phase 3 — Full Migration (Weeks 10 onwards)

### Goal
Systematically migrate all remaining services using the proven framework from Phase 2, running multiple services in parallel with confidence.

---

### Activities

#### 3.1 — Establish Migration Sprints

Organise remaining services into sprint-sized batches by risk tier. A sensible cadence looks like this:

```
Sprint 1  → 4 Green services  (pattern is proven, move fast)
Sprint 2  → 4 Green services
Sprint 3  → 3 Amber services  (slower, more engineer review)
Sprint 4  → 3 Amber services
Sprint 5  → 2 Red services    (one at a time, full team focus)
Sprint 6  → 1 Red service     (most complex, dedicated sprint)
```

---

#### 3.2 — Strangler Fig Pattern for Cutover

Never do a big-bang cutover. Use the **Strangler Fig pattern** — route traffic gradually from the legacy .NET service to the new Spring Boot service using Istio traffic splitting.

This is where your Istio investment pays off directly:

```yaml
# Istio VirtualService — gradual traffic migration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        # Week 1: 10% to new service, validate in production
        - destination:
            host: payment-service-java
            port:
              number: 8080
          weight: 10
        - destination:
            host: payment-service-dotnet
            port:
              number: 80
          weight: 90
```

You incrementally shift weight (10% → 30% → 50% → 100%) over days or weeks, monitoring for errors at each step. If anything goes wrong, you shift weight back to zero in seconds — no rollback deployment needed.

---

#### 3.3 — Continuous AI Assistance

As the team works through Amber and Red tier services, AI assistance shifts from code generation towards code review and edge case detection:

- Use Copilot to review translated business logic against the original C# and flag semantic differences
- Use Azure OpenAI to batch-scan all generated services for common anti-patterns (missing null checks, incorrect BigDecimal rounding, swallowed exceptions)
- Use Copilot to generate test cases specifically for the complex orchestration paths in Red services

---

#### 3.4 — Decommission the .NET Services

Once a service reaches 100% traffic weight on the Java side and has been stable for an agreed period (typically 2–4 weeks), decommission the .NET counterpart. Keep a documented rollback procedure for 30 days post-decommission.

---

#### Phase 3 Outputs

- All services migrated and running on AKS
- .NET application decommissioned
- Full contract test suite committed to the repository
- Migration framework documented and published as an internal reusable asset

---
---

## The Phases at a Glance

```
Week 1─3          Week 4─9              Week 10+
DISCOVERY    ──►  PILOT            ──►  FULL MIGRATION
                                        
Inventory         Common lib            Sprint batches
WSDL audit        Helm library          by risk tier
Manifest JSON     2-3 services          Strangler Fig
Risk scoring      Contract tests        Decommission
                  SAP sandbox           
                  Refine framework      
```

---

The single most important principle across all three phases is this — **the framework you build in Phase 2 is the real deliverable, not just the two pilot services**. Every hour invested in the Helm library chart, the common-lib, and the Copilot instruction file pays back multiplied across every service in Phase 3.





---

## Revised Phase Structure

| Phase | Focus | Who Leads |
|---|---|---|
| **Phase 1** | Discovery & Analysis | Architect + Senior Engineer |
| **Phase 2** | Application Migration (Spring Boot code only) | Engineering Team |
| **Phase 3** | DevOps & Infrastructure (Helm, AKS, Harness, Istio) | DevOps / Platform Team |
| **Phase 4** | Integration Testing & Cutover (SAP validation, Strangler Fig, decommission) | Engineering + DevOps together |

---

### Why This Separation Makes Sense

**Phase 2 stays purely app-focused:**
- WSDL → Java stub generation
- Business logic translation (.svc.cs → Spring Boot services)
- SAP client layer (WebServiceTemplate, Wss4j)
- REST endpoint exposure
- Unit tests and contract tests
- Common-lib module
- No Docker, no Helm, no AKS concerns at all

**Phase 3 is purely infrastructure:**
- Dockerfile per service
- Helm library chart + per-service values
- AKS configuration (namespaces, secrets, resource quotas)
- Istio VirtualService / DestinationRule templates
- Harness pipeline YAML
- Azure Key Vault integration for SAP credentials
- Container registry setup

**Phase 4 brings them together for the first time:**
- Deploy to SAP sandbox environment
- Contract tests run against real SAP
- Strangler Fig traffic shifting via Istio
- Incremental cutover service by service
- Legacy .NET decommission


Let me think through all the Phase 2 activities carefully before writing a single prompt.Now let me build the full Phase 2 instruction file.Seven clearly sequenced activities. Now let me build the full instruction file.Here's what's in the file and how it's structured.

---

### 14 Prompts Across 7 Activities

| Activity | Prompts | What it does |
|---|---|---|
| **2.1** WSDL → Java stubs | 2.1-A, 2.1-B | Generates `pom.xml` plugin config, validates JAXB output vs `Reference.cs` |
| **2.2** DTO translation | 2.2-A, 2.2-B | Translates `[DataContract]` classes to Java 21 records, single or bulk |
| **2.3** Business logic | 2.3-A, 2.3-B, 2.3-C | Analysis first, then `@Service`, then `@RestController` — deliberately sequential |
| **2.4** SAP client layer | 2.4-A, 2.4-B | `@Component` client class + `@Configuration` bean wiring with `Wss4j` |
| **2.5** Exception mapping | 2.5-A | Full exception hierarchy + static factory class replacing `FaultException<T>` |
| **2.6** Tests | 2.6-A, 2.6-B | JUnit 5 unit tests + Spring Cloud Contract stubs |
| **2.7** Common-lib | 2.7-A, 2.7-B | Candidate identification + `CorrelationIdFilter` (Istio-aware) |
| **2.R** Review gate | 2.R | Pre-merge checklist covering correctness, safety, quality, and tests |

---

### Three Design Decisions Worth Noting

**2.3-A runs before any code is generated.** The pre-translation analysis prompt forces the AI to document every business rule, SAP field mapping, and hidden complexity before touching Java. This is your insurance policy — if the generated `@Service` misses something, 2.3-A is the reference to catch it.

**The review prompt (2.R) is a hard gate.** The execution order at the bottom explicitly states no service merges without passing 2.R, and no Phase 3 starts until at least two Green-tier services are clean. This keeps AI-generated code from silently drifting into production.

**2.4-B generates its own `application.yml` snippet.** Every config class outputs the property keys it needs as a block comment — so nothing gets missed when wiring up the service for the first time.

---

Shall we now build the Phase 3 (DevOps & Infrastructure) instruction file covering Dockerfile, Helm, AKS, Istio, and Harness?





Let me think through all Phase 3 activities carefully before writing a single prompt.Good. Now let me draw the Phase 3 activity map first, then build the full instruction file.Seven layers, 14 prompts. Now building the full instruction file.Here's the full breakdown of what's in the file.

---

### 14 Prompts Across 6 Activities + Review Gate

| Activity | Prompts | What it produces |
|---|---|---|
| **3.1** Dockerfile | 3.1-A, 3.1-B | Multi-stage distroless build + `.dockerignore` |
| **3.2** ACR | 3.2-A | Registry setup, tagging strategy, Trivy scan policy, `.trivyignore` template |
| **3.3** Helm | 3.3-A, 3.3-B, 3.3-C | Library chart (shared) + per-service chart + 3 environment `values-*.yaml` |
| **3.4** AKS | 3.4-A, 3.4-B, 3.4-C | Namespaces + RBAC, Key Vault + Workload Identity, NetworkPolicies |
| **3.5** Istio | 3.5-A, 3.5-B, 3.5-C | VirtualService + DestinationRule, mTLS + AuthorizationPolicy, Gateway + ingress routing |
| **3.6** Harness | 3.6-A, 3.6-B | Shared pipeline template + thin per-service override |
| **3.R** Review gate | 3.R | 20-item checklist across security, reliability, deployment, and observability |

---

### Four Decisions Baked Into the Prompts Worth Noting

**Strangler Fig is wired in from day one.** Every `VirtualService` (3.5-A) comes with the commented-out 90/10 traffic split block pre-written and ready — you just uncomment during Phase 4 cutover. No scrambling to add it later.

**SAP retry policy is explicitly non-retrying on 5xx.** This is intentional and important — SAP SOAP operations like payment posting are often non-idempotent. Auto-retrying a failed payment would be a serious financial risk.

**The `.NET` legacy service is a first-class citizen in the Istio config.** The `dotnet-v1` subset in the `DestinationRule` and the `PERMISSIVE` mTLS override for legacy pods (3.5-B) are there from the start, so the Strangler Fig transition in Phase 4 is purely a weight change — no infrastructure rework needed mid-cutover.

**The execution order splits into a "once" block and a "per service" block.** The global infrastructure (Helm library, namespaces, Key Vault, Gateway) is built once. Individual service infra reuses it as thin overrides — exactly as intended.

Shall we build Phase 4 next — the integration testing and Strangler Fig cutover instruction file?





Let me plan Phase 4 carefully before writing a single prompt — this phase is the most operationally sensitive of all four.Good. Two diagrams for Phase 4 — the activity flow, then the Strangler Fig traffic progression. Then the full instruction file.Now the Strangler Fig traffic progression — this is the centrepiece of Phase 4 and worth its own diagram.That's the full picture. Now building the instruction file.Here's the full breakdown of what's in the file.

---

### 15 Prompts Across 7 Activities

| Activity | Prompts | What it produces |
|---|---|---|
| **4.1** SAP integration testing | 4.1-A, 4.1-B, 4.1-C | Integration test class · parity comparison script · SAP edge case tests |
| **4.2** Readiness gate | 4.2-A | 5-section go/no-go checklist with sign-off table |
| **4.3** Strangler Fig | 4.3-A, 4.3-B | Per-stage runbook · exact `kubectl patch` commands for all 4 traffic stages |
| **4.4** Production validation | 4.4-A, 4.4-B | PrometheusRule alert manifests · Grafana dashboard JSON |
| **4.5** Rollback | 4.5-A, 4.5-B | Rollback runbook · decision tree (ASCII, readable under pressure) |
| **4.6** Decommission | 4.6-A, 4.6-B | Decommission checklist · Istio cleanup patches + `.NET` removal commands |
| **4.7** Closure | 4.7-A | ADRs · operations runbook · lessons learned · framework reuse guide |

---

### Four Decisions Baked Into Phase 4 Worth Noting

**4.1-C covers the seven SAP SOAP quirks** that most commonly cause silent parity failures in `.NET` to Java migrations — whitespace padding, empty vs null fields, decimal scale, date format differences, WS-Security timestamp tolerance, large payload handling, and stateful SAP headers. These are the things the WSDL won't tell you.

**The rollback runbook (4.5-A) is written before the cutover starts**, not reactively. The critical design decision is that 4.5 is generated and reviewed in the pre-cutover window — the on-call engineer reads the decision tree before the window opens, not during an incident.

**Traffic shift commands (4.3-B) are presented as human-run commands with confirmation gates** — not automation. The global rules section explicitly prohibits any AI tool from autonomously shifting Istio weights, which is the right boundary for production financial services.

**4.7-A produces the framework reuse guide** — the final artefact that closes the loop back to your original goal. The lessons learned and reuse guide are what make this a framework for future migrations, not just a one-time exercise.

---

### The Complete Four-File Set

You now have the full framework across all four phases:

| File | Scope | Prompts |
|---|---|---|
| `phase1-copilot-instructions.md` | Discovery & analysis | 9 |
| `phase2-copilot-instructions.md` | App migration | 14 |
| `phase3-copilot-instructions.md` | DevOps & infrastructure | 14 |
| `phase4-copilot-instructions.md` | Integration & cutover | 15 |

All four files go into `.github/` in your migration repo and Copilot picks them up automatically via `@workspace`.
