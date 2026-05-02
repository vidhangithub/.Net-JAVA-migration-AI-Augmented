# Migration Copilot Instructions
# .NET WCF → Java Spring Boot — Phase 4: Integration Testing & Cutover
#
# SCOPE: SAP integration validation, Strangler Fig traffic
#        shifting, production monitoring, rollback procedures,
#        .NET decommission, and migration closure.
#        This phase brings Phase 2 (app) and Phase 3 (infra)
#        together for the first time against real SAP.
#
# CRITICAL WARNING:
#   Phase 4 operates against production SAP systems.
#   AI assistance is LIMITED to generating runbooks,
#   test scripts, monitoring queries, and checklists.
#   ALL traffic shifts and decommission actions require
#   human execution with senior engineer sign-off.
#   No AI tool should directly modify traffic weights
#   or decommission infrastructure.
#
# USAGE:
#   GitHub Copilot Chat : use @workspace for repo-aware tasks
#   Standalone LLM      : paste prompt + relevant inputs
#
# PREREQUISITE INPUTS:
#   From Phase 1:  manifest.json · risk-classification.json
#   From Phase 2:  Contract test suite passing
#   From Phase 3:  All services pass 3.R · Istio subsets live
#                  Harness pipeline deployed to staging
#
# EXECUTION ORDER:
#   4.1 → 4.2 → 4.3 → 4.4 (ongoing) → 4.5 (on-demand)
#   → 4.6 → 4.7
#   Run 4.1 and 4.2 ONCE per service before any cutover.
#   Run 4.3 per service in risk-tier order (Green first).
#   Run 4.4 continuously throughout 4.3.
#   Keep 4.5 ready at all times during 4.3.
#   Run 4.6 only after 4.4 confirms stability at 100%.
#   Run 4.7 once all services reach 4.6.
#
# CONVENTIONS:
#   [PLACEHOLDER]  → replace with your actual value
#   <<<FILE>>>     → attach or paste file content here
#   OUTPUT SCHEMA  → exact structure expected back
#
# ============================================================
# GLOBAL RULES (applied to every prompt in this file)
# ============================================================
#
# 1.  Every runbook step must include:
#     - WHO executes it (role: engineer / senior / both)
#     - WHAT the expected outcome is
#     - HOW to verify it succeeded
#     - WHAT to do if it fails (escalation or rollback step)
#
# 2.  Never produce a prompt that autonomously shifts Istio
#     traffic weights. All kubectl / helm commands that
#     change live traffic must be presented as human-run
#     commands with explicit confirmation gates.
#
# 3.  SAP field values in test data must be anonymised.
#     No real account numbers, customer IDs, or amounts
#     over 1.00 in any generated test fixture.
#
# 4.  Financial operation tests (payments, postings,
#     reversals) must include a SAP-side verification step
#     confirming no document was created in the system.
#     Use SAP sandbox only — never production SAP for tests.
#
# 5.  All generated runbook commands assume:
#     kubectl context = [AKS_CLUSTER_NAME]
#     namespace       = migration-[env]
#     Istio version   = 1.20+
#
# 6.  Observation windows are MINIMUMS not maximums.
#     Advance to the next traffic stage only when ALL
#     success criteria are met — never on a timer alone.
#
# 7.  Return only the requested YAML / shell / JSON / 
#     Markdown. No preamble unless the prompt requests
#     explanation.


# ============================================================
# ACTIVITY 4.1 — SAP INTEGRATION TESTING
# ============================================================
#
# PURPOSE:
#   Validate that every migrated service produces responses
#   that are functionally identical to the legacy .NET service
#   when calling the REAL SAP sandbox.
#   This is the highest-risk technical activity of Phase 4.
#   AI cannot execute these tests — only generate them.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.1-A  ❯  Generate SAP sandbox integration test suite
# ─────────────────────────────────────────────────────────────

PROMPT_4_1_A: |
  You are a senior Java Spring Boot integration test engineer
  with expertise in SAP SOAP services.

  Generate a Spring Boot integration test class that calls
  the REAL SAP sandbox endpoint and validates responses
  against expected behaviour documented in the Phase 2
  pre-translation analysis (2.3-A).

  RULES:
  - Test class: [ServiceName]SapIntegrationTest
  - Package: com.[ORG].[SERVICE_NAME].integration
  - Annotate with @SpringBootTest(webEnvironment = NONE)
  - Annotate with @Tag("sap-integration")
    (allows these tests to be excluded from unit test runs)
  - Use @EnabledIfEnvironmentVariable(named = "SAP_SANDBOX_URL")
    so tests only run when the sandbox is explicitly configured
  - Inject the real SapPaymentClient (not mocked)
  - SAP credentials from environment variables:
      SAP_SANDBOX_URL, SAP_SANDBOX_USER, SAP_SANDBOX_PASS

  FOR EACH SAP OPERATION IN THE SERVICE, GENERATE:

  1. Happy path test:
     - Realistic anonymised input (amounts <= 1.00 GBP)
     - Assert response is non-null
     - Assert document number is non-blank
     - Assert status field matches expected SAP success value
     - Log the SAP document number for audit trail
     - Add @AfterEach to log test duration (SAP latency matters)

  2. SAP fault test (if the operation can produce a fault):
     - Input designed to trigger a known SAP validation error
       (e.g. invalid cost centre, zero amount if SAP rejects it)
     - Assert SoapFaultClientException is thrown
     - Assert the fault detail contains the expected error code
     - Confirm the error code matches what the .NET client
       produced for the same input (document in test comment)

  3. Response parity assertion:
     - For each field in the SAP response, assert the Java
       mapping produces the same value as documented in
       the 2.3-A post_sap_logic field_mappings_from_sap
     - Flag any fields where mapping is ambiguous with:
       // PARITY CHECK: manual verification needed — <reason>

  4. Timeout behaviour test:
     - Inject a WebServiceTemplate with a 1ms timeout
     - Assert SoapFaultClientException or
       ResourceAccessException is thrown (not a silent hang)

  OUTPUT: Complete integration test class. Code only.

  PRE-TRANSLATION ANALYSIS (from Phase 2 2.3-A):
  <<<PASTE 2.3-A JSON>>>

  SAP WSDL AUDIT (from Phase 1 1.3-A):
  <<<PASTE WSDL AUDIT JSON>>>

  SERVICE DETAILS:
    name       : [SERVICE_NAME]
    port       : [SERVICE_PORT]
    sap-ops    : [SAP_OPERATION_NAMES]

# ─────────────────────────────────────────────────────────────
# PROMPT 4.1-B  ❯  Generate response parity comparison script
#               (compares .NET and Java responses side by side)
# ─────────────────────────────────────────────────────────────

PROMPT_4_1_B: |
  You are a senior integration engineer.

  Generate a shell script that calls BOTH the legacy .NET
  WCF service and the new Java Spring Boot service with
  identical inputs, captures both responses, and produces
  a field-by-field diff report.

  REQUIREMENTS:
  - Use curl for both calls
  - .NET service called via its SOAP endpoint
  - Java service called via its REST endpoint
  - Same logical input sent to both
  - Normalise field name differences (PascalCase vs camelCase)
  - Output a JSON diff report:
    {
      "test_name": "<string>",
      "timestamp": "<ISO-8601>",
      "input": { ... },
      "dotnet_response": { ... },
      "java_response": { ... },
      "field_comparison": [
        {
          "field": "<name>",
          "dotnet_value": "<string>",
          "java_value": "<string>",
          "match": <bool>,
          "note": "<null or explanation of acceptable diff>"
        }
      ],
      "overall_parity": "PASS|FAIL|REVIEW_NEEDED"
    }
  - Flag as REVIEW_NEEDED (not FAIL) when values differ
    only in formatting (e.g. trailing spaces, date format)
    and add a note explaining the difference
  - Flag as FAIL only when semantic values differ
    (document numbers, amounts, status codes)
  - Run the comparison for ALL test cases in the fixture
    file and produce a summary at the end

  OUTPUT: Bash script + a sample test-fixtures.json file
  with 3 anonymised test cases.
  Delimit with:
  # ===== FILE: parity-test.sh =====
  # ===== FILE: test-fixtures.json =====

  SERVICE DETAILS:
    dotnet-soap-url  : [DOTNET_ENDPOINT_URL]
    java-rest-url    : [JAVA_ENDPOINT_URL]
    operation        : [OPERATION_NAME]
    field-mappings   : <<<PASTE 2.3-A sap_call.field_mappings>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 4.1-C  ❯  SAP edge case test catalogue
#               (generates tests for known SAP quirks)
# ─────────────────────────────────────────────────────────────

PROMPT_4_1_C: |
  You are a senior SAP integration engineer who has
  extensive experience with SAP SOAP service quirks.

  Based on the WSDL audit and the .NET source code,
  identify and generate tests for the following categories
  of SAP edge cases that commonly cause parity failures
  during .NET to Java migrations:

  EDGE CASE CATEGORIES TO COVER:

  1. Whitespace and padding:
     SAP sometimes returns field values with trailing spaces
     or leading zeros. Test that Java handles these
     identically to the .NET client.

  2. Empty vs null fields:
     SAP may return an empty XML element vs omitting it.
     Verify the Java JAXB binding handles both and does
     not throw NullPointerException.

  3. Decimal precision:
     SAP financial amounts may use varying decimal places.
     Verify BigDecimal scale is preserved.

  4. Date/time formats:
     SAP YYYYMMDD dates must correctly parse to LocalDate.
     SAP timestamps with timezone offsets must preserve
     the original offset, not silently convert to UTC.

  5. WS-Security timestamp tolerance:
     SAP validates that the WS-Security timestamp is within
     a tolerance window (often 5 minutes). Test that the
     Java Wss4jSecurityInterceptor generates valid timestamps.

  6. Large payload handling:
     Test a response near the configured maxReceivedMessageSize
     equivalent (Spring-WS maxInMemorySize). Verify no
     truncation or OOM error.

  7. SAP session / stateful behaviour:
     Some SAP services require specific header fields to
     maintain a session. Verify the Java client sends these
     if they were present in the .NET Reference.cs.

  FOR EACH EDGE CASE:
  - Generate a JUnit 5 test method
  - Mark with @Tag("sap-edge-case")
  - Add a comment explaining the root cause and the
    .NET vs Java difference being tested
  - Flag with // MANUAL VERIFY if the edge case requires
    SAP team confirmation of expected behaviour

  OUTPUT: One test class containing all edge case tests.
  Class name: [ServiceName]SapEdgeCaseTest. Code only.

  WSDL AUDIT (from Phase 1 1.3-A):
  <<<PASTE WSDL AUDIT JSON>>>

  REFERENCE.CS KEY SECTIONS (proxy configuration):
  <<<PASTE RELEVANT Reference.cs SECTIONS>>>


# ============================================================
# ACTIVITY 4.2 — PRE-CUTOVER READINESS GATE
# ============================================================
#
# PURPOSE:
#   A structured go/no-go checklist that must be completed
#   and signed off before any Istio traffic weight is touched
#   for a given service. Run once per service, per environment.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.2-A  ❯  Generate pre-cutover readiness checklist
# ─────────────────────────────────────────────────────────────

PROMPT_4_2_A: |
  You are a senior delivery manager and migration architect.

  Generate a pre-cutover readiness checklist for a single
  migrated service, in Markdown format suitable for a
  GitHub pull request or Confluence page.

  STRUCTURE:

  Section 1 — Application readiness:
  [ ] Phase 2 code review (2.R) passed with zero blockers
  [ ] All unit tests pass (mvn test — 0 failures)
  [ ] All contract tests pass (mvn verify — 0 failures)
  [ ] SAP integration tests pass (4.1-A — 0 failures)
  [ ] Response parity check passed (4.1-B — PASS or
      all REVIEW_NEEDED items have documented acceptance)
  [ ] SAP edge case tests pass (4.1-C — 0 failures)
  [ ] All FINANCIAL: code comments reviewed by
      senior engineer [NAME] — sign-off date: ___

  Section 2 — Infrastructure readiness:
  [ ] Phase 3 infrastructure review (3.R) passed
  [ ] Service deployed to staging via Harness pipeline
  [ ] Staging smoke test passes (/actuator/health → 200)
  [ ] Istio VirtualService subsets confirmed:
        kubectl get vs [service] -n migration-staging
        both java-v1 and dotnet-v1 subsets present
  [ ] Istio DestinationRule confirmed:
        circuit breaker and outlier detection active
  [ ] Key Vault secrets accessible from staging pod:
        [provide verification command from 3.4-B OUTPUT D]
  [ ] SAP endpoint reachable from AKS namespace:
        kubectl exec test-pod -- curl -k [SAP_ENDPOINT]

  Section 3 — Observability readiness:
  [ ] Prometheus scraping service metrics (confirm in Grafana)
  [ ] Kiali shows service in mesh with healthy mTLS
  [ ] Alert rules configured for:
        error rate > 1% over 5 minutes
        p99 latency > [SAP_TIMEOUT - 5s]
        pod restart count > 2 in 10 minutes
  [ ] PagerDuty / on-call routing confirmed for prod

  Section 4 — Operational readiness:
  [ ] Rollback runbook reviewed by on-call engineer
  [ ] Change management ticket raised: CM-[NUMBER]
  [ ] Cutover window scheduled: [DATE/TIME] UTC
  [ ] SAP team notified of cutover window
  [ ] Rollback decision owner identified: [NAME/ROLE]
  [ ] Maximum acceptable error rate agreed: [THRESHOLD]%
  [ ] Observation window agreed: [HOURS] hours per stage

  Section 5 — Rollback pre-verification:
  [ ] Rollback command tested in staging:
      kubectl patch vs [service] ... (weight back to 0%)
  [ ] Confirm rollback completes within 30 seconds
  [ ] Legacy .NET service confirmed still running and
      healthy in staging

  SIGN-OFF TABLE (append at bottom):
  | Role | Name | Date | Signature |
  | Lead engineer | | | |
  | Senior engineer | | | |
  | Delivery manager | | | |
  | SAP team contact | | | |

  OUTPUT: Complete Markdown checklist.

  SERVICE DETAILS:
    name         : [SERVICE_NAME]
    environment  : [ENVIRONMENT]
    sap-endpoint : [SAP_ENDPOINT]
    sap-timeout  : [SAP_TIMEOUT_SECONDS]


# ============================================================
# ACTIVITY 4.3 — STRANGLER FIG TRAFFIC SHIFTING
# ============================================================
#
# PURPOSE:
#   Generate the exact kubectl / helm commands and
#   observation criteria for each traffic shift stage.
#   Commands are human-run — never automated.
#
# TRAFFIC STAGES (per diagram):
#   Stage 0: 100% .NET  (baseline — no action needed)
#   Stage 1:  10% Java, 90% .NET  (canary)
#   Stage 2:  50% Java, 50% .NET  (parity)
#   Stage 3: 100% Java,  0% .NET  (full cutover)
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.3-A  ❯  Generate traffic shift runbook
# ─────────────────────────────────────────────────────────────

PROMPT_4_3_A: |
  You are a senior Istio and AKS operations engineer.

  Generate a step-by-step traffic shift runbook for the
  Strangler Fig cutover of a single service.

  FOR EACH STAGE (1, 2, 3), PRODUCE:

  Step 1 — Pre-shift verification:
  - kubectl commands to confirm current weights
  - kubectl commands to confirm both pods are Running
  - Grafana / Kiali query to confirm baseline error rate
    is within acceptable threshold
  - GO condition: [list of checks that must be true]
  - NO-GO condition: [list of checks that abort the shift]

  Step 2 — Execute the weight shift:
  - kubectl patch VirtualService command setting the
    new weights (exact YAML patch — no helm here)
  - Alternative: helm upgrade command using values override
    if the team prefers Helm-driven changes
  - Confirm command to verify new weights are live:
      istioctl proxy-config routes ... or kubectl get vs

  Step 3 — Immediate post-shift verification (first 5 min):
  - kubectl top pods (watch for CPU/memory spike)
  - kubectl logs -f [java-pod] (watch for ERROR lines)
  - Grafana panel: error rate for this service
  - Grafana panel: p99 latency for this service
  - Kiali: confirm traffic is flowing to java-v1 subset
  - HOLD condition: if error rate > threshold within 5 min
    → execute rollback immediately (see 4.5 runbook)

  Step 4 — Observation window monitoring:
  - Duration: [OBSERVATION_HOURS] hours minimum
  - Metrics to watch every 30 minutes:
      error rate per operation
      p50, p95, p99 latency
      SAP fault rate
      pod restart count
  - ADVANCE condition: all metrics stable within SLO for
    the full observation window
  - ROLLBACK condition: any metric breaches threshold
    sustained for more than 5 minutes

  Step 5 — Advance decision gate:
  - Human sign-off required before next stage
  - Record: current metrics snapshot, sign-off name, time
  - Log entry to append to the cutover tracking document

  FORMAT: Markdown with collapsible sections per stage.
  Each step must include:
  WHO: [role]
  COMMAND: [exact command]
  EXPECTED: [expected output]
  IF FAILS: [action]

  SERVICE DETAILS:
    name              : [SERVICE_NAME]
    namespace         : migration-prod
    java-pod-selector : app=[SERVICE_NAME],version=java-v1
    dotnet-pod-selector: app=[SERVICE_NAME],version=dotnet-v1
    error-threshold   : [ERROR_RATE_THRESHOLD]%
    observation-hours : [OBSERVATION_HOURS]
    sap-timeout-secs  : [SAP_TIMEOUT_SECONDS]

# ─────────────────────────────────────────────────────────────
# PROMPT 4.3-B  ❯  Generate Istio weight patch commands
#               (exact commands for each stage transition)
# ─────────────────────────────────────────────────────────────

PROMPT_4_3_B: |
  You are a senior Istio engineer.

  Generate the exact kubectl patch commands to shift
  Istio VirtualService weights for each stage of the
  Strangler Fig cutover.

  PRODUCE FOUR COMMANDS:

  Command 0 — Emergency rollback (zero Java, 100% .NET):
  Used at any point if a critical issue is found.

  Command 1 — Stage 1 (10% Java, 90% .NET):
  Initial canary — only a small slice of traffic.

  Command 2 — Stage 2 (50% Java, 50% .NET):
  Parity — equal split for side-by-side comparison.

  Command 3 — Stage 3 (100% Java, 0% .NET):
  Full cutover — all traffic on Java.

  FORMAT FOR EACH COMMAND:
  # ── STAGE [N]: [description] ──
  # WHO:      [role]
  # CONFIRM:  [pre-condition check command]
  # EXECUTE:
  kubectl patch virtualservice [service-name] \
    -n [namespace] \
    --type=merge \
    -p '[exact JSON patch]'
  # VERIFY:
  kubectl get virtualservice [service-name] \
    -n [namespace] -o jsonpath='...'
  # EXPECTED: [exact expected output]

  Also generate a single helm upgrade alternative for
  teams that prefer Helm-driven changes over raw kubectl:
  helm upgrade [service] ./[service] \
    -f values-prod.yaml \
    --set istio.javaWeight=[N] \
    --set istio.dotnetWeight=[M] \
    -n [namespace]

  OUTPUT: Shell script with all four commands.
  File header must include:
  # WARNING: These commands modify live production traffic.
  # Each command requires senior engineer sign-off.
  # Run only during the agreed cutover window.

  SERVICE DETAILS:
    name      : [SERVICE_NAME]
    namespace : migration-prod


# ============================================================
# ACTIVITY 4.4 — PRODUCTION VALIDATION
# ============================================================
#
# PURPOSE:
#   Generate the monitoring queries, alert rules, and
#   SLO definitions used throughout the Strangler Fig
#   observation windows. These run continuously from
#   Stage 1 onwards.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.4-A  ❯  Generate Prometheus alert rules
# ─────────────────────────────────────────────────────────────

PROMPT_4_4_A: |
  You are a senior observability engineer with expertise
  in Prometheus, Istio telemetry, and Spring Boot Actuator.

  Generate PrometheusRule manifests for the cutover
  observation windows of a migrated service.

  GENERATE ALERT RULES FOR:

  1. Error rate spike (critical):
  - Metric: rate of 5xx responses from java-v1 subset
    over 5-minute window
  - Threshold: > [ERROR_RATE_THRESHOLD]% sustained 5 min
  - Severity: critical
  - Annotation: "Possible rollback required — check 4.5"

  2. SAP fault rate (warning):
  - Metric: rate of SoapFaultClientException from
    Micrometer counter (tag: exception=SoapFaultClientException)
  - Threshold: > [SAP_FAULT_THRESHOLD] per minute
  - Severity: warning
  - Annotation: "SAP returning faults — check SAP sandbox logs"

  3. Latency regression (warning):
  - Metric: histogram_quantile(0.99, ...) for java-v1
    compared to dotnet-v1 p99
  - Threshold: Java p99 > .NET p99 * 1.5 sustained 10 min
  - Severity: warning
  - Annotation: "Java p99 significantly higher than .NET baseline"

  4. Pod restart (critical):
  - Metric: kube_pod_container_status_restarts_total
    for java-v1 pods
  - Threshold: increase > 2 in 10-minute window
  - Severity: critical
  - Annotation: "Pod crash loop — immediate investigation needed"

  5. Istio circuit breaker open (warning):
  - Metric: envoy_cluster_outlier_detection_ejections_active
    for java-v1 upstream cluster
  - Threshold: > 0
  - Severity: warning
  - Annotation: "Circuit breaker has ejected pods — check pod health"

  6. Weight drift detection (info):
  - Metric: Confirm VirtualService weights match intended
    stage (custom metric or periodic kubectl check)
  - Alert if weights differ from last recorded stage
  - Severity: info
  - Annotation: "VirtualService weights changed unexpectedly"

  OUTPUT: Complete PrometheusRule YAML manifest.
  Include the Phase 4 header comment.

  DETAILS:
    service-name          : [SERVICE_NAME]
    namespace             : migration-prod
    error-rate-threshold  : [ERROR_RATE_THRESHOLD]
    sap-fault-threshold   : [SAP_FAULT_THRESHOLD]

# ─────────────────────────────────────────────────────────────
# PROMPT 4.4-B  ❯  Generate Grafana dashboard JSON
# ─────────────────────────────────────────────────────────────

PROMPT_4_4_B: |
  You are a senior Grafana and Istio observability engineer.

  Generate a Grafana dashboard JSON (v9+ format) for the
  Strangler Fig cutover monitoring of a single service.

  DASHBOARD PANELS:

  Row 1 — Traffic split (live):
  - Pie chart: current traffic % to java-v1 vs dotnet-v1
    (from Istio telemetry: istio_requests_total by version)
  - Stat panel: requests/sec to java-v1
  - Stat panel: requests/sec to dotnet-v1

  Row 2 — Error rates:
  - Time series: error rate java-v1 (5xx / total)
  - Time series: error rate dotnet-v1 (5xx / total)
  - Threshold lines at [ERROR_RATE_THRESHOLD]%

  Row 3 — Latency:
  - Time series: p50, p95, p99 latency java-v1
  - Time series: p50, p95, p99 latency dotnet-v1
  - Annotation: vertical line at each traffic weight change

  Row 4 — SAP integration:
  - Time series: SAP fault rate (java-v1)
  - Time series: SAP call duration p99 (java-v1)
  - Stat panel: total SAP documents created (counter)

  Row 5 — Infrastructure:
  - Time series: pod CPU usage java-v1
  - Time series: pod memory usage java-v1
  - Stat panel: pod restart count java-v1

  DASHBOARD SETTINGS:
  - Refresh: 30s
  - Time range: last 6 hours default
  - Tags: migration, [service-name], strangler-fig

  OUTPUT: Complete Grafana dashboard JSON.
  Replace metric names with correct Istio 1.20+ and
  Spring Boot 3.x Actuator / Micrometer metric names.

  DETAILS:
    service-name          : [SERVICE_NAME]
    namespace             : migration-prod
    error-rate-threshold  : [ERROR_RATE_THRESHOLD]


# ============================================================
# ACTIVITY 4.5 — ROLLBACK PROCEDURES
# ============================================================
#
# PURPOSE:
#   Pre-written rollback runbook and decision tree.
#   Must be reviewed and understood by the on-call engineer
#   BEFORE any traffic is shifted. Not written reactively.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.5-A  ❯  Generate rollback runbook
# ─────────────────────────────────────────────────────────────

PROMPT_4_5_A: |
  You are a senior site reliability engineer.

  Generate a rollback runbook for a Strangler Fig cutover.
  This document is read under pressure — it must be
  unambiguous, step-by-step, and executable in under
  2 minutes by any on-call engineer.

  STRUCTURE:

  Section 1 — When to roll back (decision criteria):
  Immediate rollback (no discussion needed):
  - Error rate > [ERROR_RATE_THRESHOLD]% for 5+ minutes
  - Pod crash loop (restarts > 2 in 10 minutes)
  - SAP returning faults at > [SAP_FAULT_THRESHOLD]/min
  - Any financial discrepancy reported by SAP team
  - Istio circuit breaker open on java-v1

  Hold and assess (5-minute window before deciding):
  - Latency increase < 50% above baseline
  - Single isolated error spike (< 2 minutes)
  - Intermittent warning alerts without sustained trend

  Do NOT roll back for:
  - Increased log verbosity (expected during observation)
  - Slightly higher CPU usage (expected at new traffic levels)
  - Alert noise from misconfigured thresholds

  Section 2 — Rollback execution (step by step):
  [Use exact kubectl patch command from 4.3-B Command 0]
  Each step: WHO / COMMAND / EXPECTED / IF FAILS

  Section 3 — Post-rollback actions:
  - Verify traffic back on .NET (kubectl get vs)
  - Verify error rate returns to baseline within 2 minutes
  - Page senior engineer if not resolved within 5 minutes
  - Open incident ticket — template provided
  - Notify SAP team if any SAP document may be affected
  - Do NOT re-attempt cutover until root cause is confirmed

  Section 4 — Incident ticket template:
  Title : [SERVICE_NAME] cutover rollback — [DATE]
  Stage : Was at Stage [N] when rollback triggered
  Trigger: [which alert or observation caused rollback]
  Impact : [number of requests affected, error rate peak]
  Timeline: [timestamps of shift, detection, rollback]
  RCA    : [to be completed post-incident]
  Next steps: [to be agreed with team before retry]

  OUTPUT: Markdown runbook. Suitable for Confluence or
  GitHub wiki. Include warning callout boxes for the
  most critical steps.

  SERVICE DETAILS:
    name                 : [SERVICE_NAME]
    namespace            : migration-prod
    error-threshold      : [ERROR_RATE_THRESHOLD]%
    sap-fault-threshold  : [SAP_FAULT_THRESHOLD]/min
    on-call-contact      : [ON_CALL_CONTACT]
    sap-team-contact     : [SAP_TEAM_CONTACT]

# ─────────────────────────────────────────────────────────────
# PROMPT 4.5-B  ❯  Generate rollback decision tree
# ─────────────────────────────────────────────────────────────

PROMPT_4_5_B: |
  You are a senior SRE and delivery engineer.

  Generate a plain-text decision tree (using ASCII art /
  indented structure) that an on-call engineer can follow
  in real time during a cutover incident.

  The tree must cover:
  - Start: "Alert received during cutover"
  - Branch: Is error rate > threshold?
  - Branch: Is it sustained (> 5 min)?
  - Branch: Is it a SAP fault or an application error?
  - Branch: Is the legacy .NET service still healthy?
  - Terminal actions:
      ROLLBACK NOW (with command reference)
      HOLD AND MONITOR (with next check time)
      ESCALATE TO SENIOR ENGINEER
      CONTACT SAP TEAM

  Keep each decision node to one yes/no question.
  Each terminal action references the exact runbook
  section from 4.5-A.

  OUTPUT: Plain text decision tree. No Markdown tables.
  Wrap at 80 characters. Use indentation and ASCII for
  branching.

  SERVICE DETAILS:
    name             : [SERVICE_NAME]
    error-threshold  : [ERROR_RATE_THRESHOLD]%
    escalation-chain : [NAMES/ROLES IN ORDER]


# ============================================================
# ACTIVITY 4.6 — .NET SERVICE DECOMMISSION
# ============================================================
#
# PURPOSE:
#   Structured checklist and removal sequence for the legacy
#   .NET WCF service. Run only after:
#   - Traffic has been at 100% Java for [STABILITY_DAYS] days
#   - No rollback has been triggered in that period
#   - Senior engineer and SAP team have signed off
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.6-A  ❯  Generate decommission readiness checklist
# ─────────────────────────────────────────────────────────────

PROMPT_4_6_A: |
  You are a senior migration architect.

  Generate a .NET service decommission readiness checklist
  for a single migrated service.

  STRUCTURE:

  Section 1 — Stability confirmation:
  [ ] Java service has been at 100% traffic weight for
      [STABILITY_DAYS] days with no rollback
  [ ] Error rate has been within SLO every day of that
      period (attach Grafana screenshot)
  [ ] SAP team confirms no unexpected documents or
      anomalies in the [STABILITY_DAYS]-day window
  [ ] All FINANCIAL: flagged operations reviewed by
      finance team — sign-off date: ___
  [ ] No open incidents related to this service

  Section 2 — Rollback option removal:
  [ ] Decision confirmed: removing rollback capability
      once dotnet-v1 subset is removed
  [ ] Final rollback test executed in staging
      (confirm it still works before removing production option)
  [ ] Change management ticket for decommission: CM-[NUMBER]

  Section 3 — Istio cleanup:
  [ ] Remove dotnet-v1 subset from DestinationRule
  [ ] Remove dotnet-v1 route from VirtualService
  [ ] Remove PeerAuthentication PERMISSIVE override
      (the temporary one for legacy pods from 3.5-B)
  [ ] Confirm mTLS STRICT now applies to all pods

  Section 4 — .NET service removal:
  [ ] Scale .NET deployment to 0 replicas first (dry run)
  [ ] Monitor for 24 hours — confirm no traffic or errors
  [ ] Delete .NET deployment
  [ ] Delete .NET service (Kubernetes Service resource)
  [ ] Remove .NET-related secrets and configmaps
  [ ] Archive .NET source code (do not delete — tag as ARCHIVED)

  Section 5 — Infrastructure cleanup:
  [ ] Remove web.config and IIS configuration from docs
  [ ] Remove .NET Docker image from ACR
      (retain for 30 days in case rollback is re-evaluated)
  [ ] Remove .NET-specific Harness pipeline stage
  [ ] Update network policies to remove .NET egress rules

  Section 6 — Documentation:
  [ ] Architecture diagram updated to remove .NET service
  [ ] Runbook updated — remove all rollback-to-.NET steps
  [ ] ADR written: why this service was migrated and when

  OUTPUT: Complete Markdown checklist with sign-off table.

  SERVICE DETAILS:
    name            : [SERVICE_NAME]
    stability-days  : [STABILITY_DAYS]
    sap-team-contact: [SAP_TEAM_CONTACT]

# ─────────────────────────────────────────────────────────────
# PROMPT 4.6-B  ❯  Generate Istio cleanup manifest patches
# ─────────────────────────────────────────────────────────────

PROMPT_4_6_B: |
  You are a senior Istio engineer.

  Generate the exact kubectl patch commands and updated
  YAML manifests to clean up all Istio resources after
  the .NET service has been decommissioned.

  PRODUCE:

  1. DestinationRule patch — remove dotnet-v1 subset:
     kubectl patch command using --type=json patch
     to remove only the dotnet-v1 entry from subsets array
     while preserving java-v1 and all trafficPolicy config

  2. VirtualService patch — remove dotnet-v1 route:
     kubectl patch to remove dotnet-v1 destination
     and set java-v1 weight to 100 (if not already)
     Remove the commented-out Strangler Fig block

  3. PeerAuthentication patch — remove PERMISSIVE override:
     kubectl delete the per-pod PeerAuthentication that
     had mode: PERMISSIVE for dotnet-v1 pods

  4. Verify STRICT mTLS is now enforced everywhere:
     istioctl authn tls-check command to confirm all
     pods in namespace are using STRICT mTLS

  5. Final state YAML for both DR and VS:
     Show the clean post-decommission version of both
     resources as reference for the GitOps repo

  Each command in format:
  # WHO: [role]
  # CONFIRM: [pre-condition]
  # EXECUTE: [command]
  # VERIFY: [verification command + expected output]

  SERVICE DETAILS:
    name      : [SERVICE_NAME]
    namespace : migration-prod


# ============================================================
# ACTIVITY 4.7 — MIGRATION CLOSURE
# ============================================================
#
# PURPOSE:
#   Document the migration, capture lessons learned, and
#   publish the framework for reuse on future migrations.
#   Run once after ALL services are decommissioned.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 4.7-A  ❯  Generate migration closure documentation
# ─────────────────────────────────────────────────────────────

PROMPT_4_7_A: |
  You are a senior technical writer and migration architect.

  Generate the migration closure documentation package
  for the completed .NET WCF to Java Spring Boot migration.

  PRODUCE FOUR DOCUMENTS:

  DOCUMENT A — Architecture Decision Records (ADR) template:
  One ADR per major decision made during the migration.
  Use the standard ADR format:
    Title, Date, Status, Context, Decision, Consequences
  Generate ADR stubs for these decisions (team fills detail):
  - ADR-001: Why Java 21 records for DTOs
  - ADR-002: Why Spring-WS over JAX-WS
  - ADR-003: Strangler Fig vs big-bang cutover decision
  - ADR-004: Inter-service call strategy
    (REST vs direct calls — from 1.2 dependency graph)
  - ADR-005: Common-lib boundary decisions
  - ADR-006: Istio mTLS STRICT enforcement timing

  DOCUMENT B — Operations runbook (post-migration):
  Concise runbook for the Java services now in production:
  - How to deploy a new version (Harness pipeline steps)
  - How to roll back a deployment (helm rollback)
  - How to check SAP connectivity from a pod
  - How to rotate SAP credentials (Key Vault + pod restart)
  - How to scale a service manually (kubectl scale)
  - How to read Istio telemetry in Kiali
  - Common error codes and their SAP root causes
    (document the error codes found during 4.1-A tests)

  DOCUMENT C — Lessons learned (template):
  Structured lessons learned document with sections:
  - What worked well (prompt the team to fill in)
  - What was harder than expected
  - SAP integration surprises
  - AI tool effectiveness per phase (Copilot, Claude)
  - Framework improvements for next migration
  - Time estimates vs actuals per phase

  DOCUMENT D — Framework reuse guide:
  How to use this 4-phase framework for future migrations.
  Sections:
  - Which artefacts are reusable as-is (Helm library,
    Copilot instruction files, common-lib)
  - Which artefacts need customisation (WSDL-specific
    config, service-specific values.yaml)
  - How to adapt Phase 1 discovery for a non-SAP target
  - How to adapt Phase 2 for a non-WCF source
    (e.g. REST-to-REST, Python-to-Java)
  - Recommended team structure and skill requirements

  OUTPUT: Four Markdown documents, delimited with:
  # ===== DOCUMENT A: ADRs =====
  # ===== DOCUMENT B: Operations Runbook =====
  # ===== DOCUMENT C: Lessons Learned =====
  # ===== DOCUMENT D: Framework Reuse Guide =====

  MIGRATION SUMMARY INPUTS:
    total-services    : [TOTAL_SERVICES_MIGRATED]
    green-count       : [GREEN_COUNT]
    amber-count       : [AMBER_COUNT]
    red-count         : [RED_COUNT]
    total-sprints     : [ACTUAL_SPRINTS]
    team-size         : [TEAM_SIZE]
    sap-operations    : [TOTAL_SAP_OPERATIONS]
    shared-utilities  : <<<PASTE shared_utilities from manifest.json>>>


# ============================================================
# PHASE 4 — EXECUTION ORDER SUMMARY
# ============================================================
#
# PER SERVICE (in risk-tier order — Green first):
#
#   4.1-A  → SAP integration test class
#   4.1-B  → Response parity comparison script
#   4.1-C  → SAP edge case test catalogue
#            ↓ all three must PASS before continuing
#   4.2-A  → Pre-cutover readiness checklist
#            ↓ all checklist items ticked + signed off
#   4.3-B  → Generate traffic shift commands
#            (review these BEFORE the cutover window)
#   4.5-A  → Rollback runbook
#   4.5-B  → Rollback decision tree
#            ↓ on-call engineer confirms both are understood
#   4.4-A  → Deploy PrometheusRule alert manifests
#   4.4-B  → Import Grafana dashboard
#            ↓ confirm alerts are firing in staging test
#
#   [CUTOVER WINDOW BEGINS]
#   4.3-A  → Execute Stage 1 (10%) using 4.3-B commands
#            → observe for [OBSERVATION_HOURS] hours
#   4.3-A  → Execute Stage 2 (50%) if Stage 1 criteria met
#            → observe for [OBSERVATION_HOURS] hours
#   4.3-A  → Execute Stage 3 (100%) if Stage 2 criteria met
#            → observe for [STABILITY_DAYS] days
#   [CUTOVER WINDOW ENDS — service is now fully on Java]
#
#   4.6-A  → Decommission readiness checklist
#            ↓ all items ticked + signed off
#   4.6-B  → Execute Istio cleanup + .NET removal
#
# ONCE (after all services are complete):
#   4.7-A  → Migration closure documentation
#
# MIGRATION IS COMPLETE WHEN:
#   ✓ All services at 100% Java traffic for [STABILITY_DAYS]
#   ✓ All .NET deployments deleted
#   ✓ Istio STRICT mTLS confirmed on all pods (no PERMISSIVE)
#   ✓ All ADRs written and merged to repo
#   ✓ Operations runbook published
#   ✓ Lessons learned session completed
#   ✓ Framework reuse guide committed to inner-source repo
# ============================================================
