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

Would you like to go deeper on any specific phase — for example, the Strangler Fig Istio config, the contract test setup, or how to structure the Harness pipeline template?
