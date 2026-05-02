# Migration Copilot Instructions
# .NET WCF → Java Spring Boot — Phase 1: Discovery & Analysis
#
# USAGE:
#   GitHub Copilot Chat : paste the relevant prompt block into the chat panel
#                         prefix with @workspace when running inside the IDE
#                         against the legacy .NET solution
#   Standalone LLM      : paste the prompt block directly; attach or paste
#                         the relevant source file(s) as context
#
# CONVENTIONS USED IN THIS FILE:
#   [PLACEHOLDER]  → replace with your actual value before sending
#   <<<FILE>>>     → attach or paste the file content at that point
#   OUTPUT SCHEMA  → the exact JSON / Markdown structure expected back

# ============================================================
# GLOBAL RULES (applied to every prompt in this file)
# ============================================================
#
# 1. Never invent values. If a field cannot be determined from
#    the source, set it to null and add a "review_flag": true.
# 2. Always preserve original .NET identifiers (class names,
#    method names, namespace paths) in a "legacy_name" field
#    so nothing gets lost in translation.
# 3. Flag anything financially sensitive with
#    "financial_sensitivity": true — these need senior engineer
#    sign-off before the migrated code is merged.
# 4. When confidence is below ~80%, add a
#    "low_confidence_reason": "<explanation>" field rather than
#    guessing silently.
# 5. Return strictly valid JSON or Markdown as specified.
#    No preamble, no trailing commentary.

# ============================================================
# ACTIVITY 1.1 — CODEBASE & WCF SERVICE INVENTORY
# ============================================================
#
# PURPOSE:
#   Produce a structured JSON inventory of every WCF service
#   in the solution. This becomes the migration manifest that
#   drives all subsequent phases.
#
# WHEN TO USE:
#   Run once per .NET solution / project folder.
#   Re-run if new .svc files are discovered later.
#
# INPUTS REQUIRED:
#   - Full solution folder (via @workspace in Copilot)
#     OR paste individual .svc.cs + web.config content below
#
# ─────────────────────────────────────────────────────────────
# PROMPT 1.1-A  ❯  Full Solution Scan  (@workspace / folder)
# ─────────────────────────────────────────────────────────────

PROMPT_1_1_A: |
  @workspace

  You are a senior migration analyst.
  Scan the entire .NET solution and produce a JSON service
  inventory following the OUTPUT SCHEMA below.

  RULES:
  - Find every file ending in .svc or .svc.cs
  - Find every [ServiceContract] interface
  - Find every [OperationContract] method
  - Find every [DataContract] / [DataMember] class
  - Find every auto-generated SAP proxy (Reference.cs or
    files containing ClientBase<T>)
  - Extract all <endpoint> entries from web.config /
    app.config that reference SAP hosts
  - Identify shared utilities referenced across multiple
    services (helpers, loggers, validators)
  - Apply GLOBAL RULES above

  OUTPUT SCHEMA:
  {
    "scan_metadata": {
      "solution_name": "<string>",
      "scan_date": "<ISO-8601>",
      "total_services_found": <int>
    },
    "services": [
      {
        "service_id": "<short-kebab-id>",
        "legacy_name": "<ClassName from .svc.cs>",
        "svc_file": "<relative path>",
        "interface_file": "<relative path to ServiceContract>",
        "namespace": "<C# namespace>",
        "operations": [
          {
            "legacy_name": "<OperationContract method name>",
            "input_type": "<DataContract class name>",
            "output_type": "<DataContract class name>",
            "fault_types": ["<FaultContract type>"],
            "financial_sensitivity": <bool>,
            "review_flag": <bool>
          }
        ],
        "sap_proxy": {
          "generated_file": "<Reference.cs path or null>",
          "wsdl_source": "<WSDL filename or URL if visible>",
          "sap_operations_called": ["<method names on ClientBase>"]
        },
        "web_config": {
          "endpoint_name": "<string>",
          "address": "<URL>",
          "binding": "<basicHttpBinding|wsHttpBinding|etc>",
          "security_mode": "<None|Transport|Message|TransportWithMessageCredential>",
          "credential_type": "<UserName|Certificate|Windows|null>",
          "send_timeout": "<duration string>",
          "receive_timeout": "<duration string>",
          "max_message_size_bytes": <int or null>
        },
        "shared_dependencies": ["<class or utility name>"],
        "low_confidence_reason": "<string or null>"
      }
    ],
    "shared_utilities": [
      {
        "name": "<class name>",
        "file": "<relative path>",
        "used_by_services": ["<service_id>"],
        "description": "<one-line purpose>"
      }
    ]
  }

# ─────────────────────────────────────────────────────────────
# PROMPT 1.1-B  ❯  Single Service Deep Dive
#               (use when 1.1-A flags low_confidence on a service)
# ─────────────────────────────────────────────────────────────

PROMPT_1_1_B: |
  You are a senior migration analyst.

  Below is a single WCF service implementation.
  Produce a detailed JSON entry conforming to the service
  object schema in PROMPT_1_1_A.

  Pay special attention to:
  - Any business logic BEFORE the SAP call (validation,
    enrichment, lookups) — list each step under
    "pre_sap_logic" as a plain-English bullet
  - Any business logic AFTER the SAP response (mapping,
    transformation, audit) — list under "post_sap_logic"
  - Any hardcoded values (magic strings, default currencies,
    cost centres) — list under "hardcoded_values" with the
    line reference
  - Any calls to other internal WCF services (not SAP) —
    list under "internal_service_calls"

  Extend the service schema with these additional fields:
  {
    "pre_sap_logic": ["<plain English step>"],
    "post_sap_logic": ["<plain English step>"],
    "hardcoded_values": [
      { "value": "<string>", "line_ref": "<approx line or method>" }
    ],
    "internal_service_calls": [
      { "target_service": "<name>", "operation": "<method>" }
    ]
  }

  SOURCE FILE:
  <<<PASTE .svc.cs CONTENT HERE>>>

  WEB.CONFIG SNIPPET:
  <<<PASTE RELEVANT <system.serviceModel> BLOCK HERE>>>


# ============================================================
# ACTIVITY 1.2 — DEPENDENCY GRAPH MAPPING
# ============================================================
#
# PURPOSE:
#   Produce a dependency graph showing which services call
#   each other (internal) and which call SAP (external).
#   Drives microservice boundary decisions.
#
# WHEN TO USE:
#   After 1.1 is complete. Feed the 1.1 manifest JSON as
#   input to prompt 1.2-A.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 1.2-A  ❯  Generate Dependency Graph from Manifest
# ─────────────────────────────────────────────────────────────

PROMPT_1_2_A: |
  You are a senior solution architect.

  Below is a migration manifest JSON produced by a WCF
  codebase scan (see PROMPT_1_1_A output).

  Produce TWO outputs:

  OUTPUT A — JSON dependency graph:
  {
    "nodes": [
      {
        "id": "<service_id>",
        "type": "internal|sap|shared_utility",
        "label": "<human readable name>"
      }
    ],
    "edges": [
      {
        "from": "<service_id or utility name>",
        "to": "<service_id or SAP operation>",
        "type": "internal_call|sap_call|utility_usage",
        "operations": ["<operation names>"],
        "notes": "<any coupling concern>"
      }
    ],
    "circular_dependencies": ["<service_id pair if found>"],
    "consolidation_candidates": [
      {
        "services": ["<service_id>"],
        "reason": "<why these could be one microservice>"
      }
    ],
    "split_candidates": [
      {
        "service": "<service_id>",
        "reason": "<why this should become multiple microservices>"
      }
    ]
  }

  OUTPUT B — Plain English summary (max 200 words):
  Describe the dependency landscape, highlight any tight
  coupling risks, circular dependencies, and your top
  recommendation for microservice boundaries.

  MANIFEST JSON:
  <<<PASTE 1.1-A OUTPUT HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 1.2-B  ❯  Inter-Service Call Detection
#               (use when @workspace scan missed internal calls)
# ─────────────────────────────────────────────────────────────

PROMPT_1_2_B: |
  @workspace

  Scan the entire solution for any place where one WCF
  service implementation directly instantiates or injects
  another WCF client proxy (i.e. a class extending
  ClientBase<T> that is NOT a SAP proxy).

  For each internal call found, return:
  {
    "caller_service": "<legacy class name>",
    "caller_file": "<relative path>",
    "callee_service": "<legacy class name>",
    "callee_file": "<relative path>",
    "call_site_method": "<method where the call happens>",
    "call_purpose": "<one-line description>",
    "migration_concern": "<what needs deciding at design time>"
  }

  Return as a JSON array. If none found, return [].


# ============================================================
# ACTIVITY 1.3 — SAP WSDL AUDIT
# ============================================================
#
# PURPOSE:
#   Audit every SAP WSDL the application consumes.
#   Determine what security, types, and operations are
#   involved so the Spring Boot SAP client layer can be
#   planned accurately.
#
# WHEN TO USE:
#   For each WSDL file collected from the .NET project
#   (typically found in Service References folders or as
#   standalone .wsdl files).
#
# ─────────────────────────────────────────────────────────────
# PROMPT 1.3-A  ❯  Single WSDL Deep Audit
# ─────────────────────────────────────────────────────────────

PROMPT_1_3_A: |
  You are a senior integration architect specialising in
  SAP SOAP services and Java Spring-WS.

  Analyse the WSDL below and produce a JSON audit report.

  RULES:
  - Identify every operation (portType/operation)
  - Identify every input/output/fault message type
  - Identify the binding style (document/literal vs rpc)
  - Identify the security scheme from the binding or any
    WS-Policy attachments
  - Flag any non-standard SAP extensions or custom headers
  - Assess Java migration complexity for each operation
  - For each XSD type, state the recommended Java mapping
    (record, class, enum, BigDecimal for decimal, etc.)

  OUTPUT SCHEMA:
  {
    "wsdl_name": "<filename>",
    "target_namespace": "<string>",
    "sap_endpoint_url": "<from soap:address>",
    "binding_style": "document-literal|rpc-literal|rpc-encoded",
    "ws_security": {
      "scheme": "UsernameToken|Certificate|None|Unknown",
      "transport": "HTTPS|HTTP",
      "notes": "<any policy attachment details>"
    },
    "operations": [
      {
        "name": "<operation name>",
        "soap_action": "<string>",
        "input_message": "<message name>",
        "output_message": "<message name>",
        "fault_messages": ["<fault message name>"],
        "java_method_signature": "<suggested Java method sig>",
        "migration_complexity": "low|medium|high",
        "complexity_reason": "<string or null>"
      }
    ],
    "xsd_types": [
      {
        "name": "<complexType or element name>",
        "java_mapping": "<recommended Java type>",
        "fields": [
          {
            "name": "<field name>",
            "xsd_type": "<xsd:string etc>",
            "java_type": "<String, BigDecimal, etc>",
            "nullable": <bool>,
            "financial_sensitivity": <bool>
          }
        ]
      }
    ],
    "non_standard_extensions": ["<description>"],
    "spring_ws_notes": "<specific Spring-WS config implications>",
    "jaxb_generation_flags": "<any special xjc binding flags needed>"
  }

  WSDL CONTENT:
  <<<PASTE WSDL XML HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 1.3-B  ❯  WSDL vs Reference.cs Drift Detection
#               (detects if the .NET proxy is out of sync
#                with the actual SAP WSDL)
# ─────────────────────────────────────────────────────────────

PROMPT_1_3_B: |
  You are a senior integration engineer.

  Compare the SAP WSDL (SOURCE OF TRUTH) against the
  auto-generated .NET proxy file (Reference.cs).

  Identify any drift — operations, fields, or types present
  in one but not the other, or where types differ.

  OUTPUT SCHEMA:
  {
    "drift_detected": <bool>,
    "summary": "<one paragraph>",
    "missing_in_proxy": [
      {
        "item": "<operation or type name>",
        "in_wsdl": true,
        "in_proxy": false,
        "impact": "<what this means for migration>"
      }
    ],
    "missing_in_wsdl": [
      {
        "item": "<operation or type name>",
        "in_wsdl": false,
        "in_proxy": true,
        "impact": "<possibly obsolete — confirm with SAP team>"
      }
    ],
    "type_mismatches": [
      {
        "field": "<field path>",
        "wsdl_type": "<xsd type>",
        "proxy_type": "<C# type>",
        "recommended_java_type": "<string>",
        "risk": "low|medium|high"
      }
    ],
    "recommendation": "<regenerate from WSDL|use proxy as-is|manual merge>"
  }

  WSDL CONTENT:
  <<<PASTE WSDL XML HERE>>>

  REFERENCE.CS CONTENT:
  <<<PASTE Reference.cs CONTENT HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 1.3-C  ❯  WSDL Registry Builder
#               (run after all individual 1.3-A audits are done)
# ─────────────────────────────────────────────────────────────

PROMPT_1_3_C: |
  You are a senior migration analyst.

  Below are multiple WSDL audit reports (1.3-A outputs).
  Consolidate them into a single WSDL registry.

  Identify:
  - WSDLs that share the same SAP host (can share one
    WebServiceTemplate bean)
  - WSDLs that use the same security scheme (can share one
    Wss4jSecurityInterceptor bean)
  - WSDLs that need certificate-based auth (flag for Azure
    Key Vault certificate setup)
  - Any duplicate type definitions across WSDLs (candidates
    for a shared JAXB model module)

  OUTPUT SCHEMA:
  {
    "registry": [
      {
        "wsdl_name": "<string>",
        "sap_host": "<hostname only>",
        "security_scheme": "<string>",
        "operations_count": <int>,
        "shared_bean_group": "<group-id for shared Spring beans>",
        "key_vault_cert_required": <bool>
      }
    ],
    "shared_bean_groups": [
      {
        "group_id": "<string>",
        "wsdls": ["<wsdl_name>"],
        "spring_bean_strategy": "<description of shared config>"
      }
    ],
    "shared_type_candidates": [
      {
        "type_name": "<string>",
        "found_in_wsdls": ["<wsdl_name>"],
        "recommendation": "extract to shared-model module|keep separate"
      }
    ]
  }

  WSDL AUDIT REPORTS:
  <<<PASTE ALL 1.3-A OUTPUTS AS A JSON ARRAY HERE>>>


# ============================================================
# ACTIVITY 1.4 — RISK CLASSIFICATION
# ============================================================
#
# PURPOSE:
#   Score every service for migration complexity so the
#   team can sequence the work correctly (Green first,
#   Red last).
#
# WHEN TO USE:
#   After 1.1, 1.2, and 1.3 are complete.
#   Feed all three outputs together for the most accurate
#   classification.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 1.4-A  ❯  Risk Score Each Service
# ─────────────────────────────────────────────────────────────

PROMPT_1_4_A: |
  You are a senior Java migration architect.

  Using the inputs below, classify every service in the
  migration manifest by migration risk tier.

  SCORING CRITERIA — add points for each factor present:

  Complexity factors (code):
    +1  Has 1–2 SAP operations
    +2  Has 3–5 SAP operations
    +3  Has 6+ SAP operations
    +2  Has internal calls to other WCF services
    +2  Has pre/post SAP business logic beyond simple mapping
    +3  Has financial calculations (rounding, currency, tax)
    +2  Has hardcoded values that need externalising
    +1  Has shared utility dependencies

  Integration factors (SAP):
    +1  Uses basicHttpBinding / UsernameToken (well understood)
    +2  Uses wsHttpBinding / WS-Security certificates
    +3  Has WSDL drift detected (1.3-B flagged issues)
    +2  Has non-standard SAP extensions
    +1  Has large message sizes (>1MB)

  Operational factors:
    +2  Service is on a critical financial transaction path
    +1  Service has no existing automated tests in .NET
    +2  Service has no SAP sandbox available for testing

  RISK TIERS:
    Green  = 0–4   (migrate in Phase 3 Sprint 1–2, AI-led)
    Amber  = 5–9   (migrate in Phase 3 Sprint 3–4, engineer-led with AI assist)
    Red    = 10+   (migrate in Phase 3 Sprint 5–6, senior engineer, full review)

  OUTPUT SCHEMA:
  {
    "classification_date": "<ISO-8601>",
    "services": [
      {
        "service_id": "<string>",
        "legacy_name": "<string>",
        "score": <int>,
        "tier": "Green|Amber|Red",
        "score_breakdown": {
          "complexity_factors": [
            { "factor": "<description>", "points": <int> }
          ],
          "integration_factors": [
            { "factor": "<description>", "points": <int> }
          ],
          "operational_factors": [
            { "factor": "<description>", "points": <int> }
          ]
        },
        "top_risks": ["<plain English risk statement>"],
        "recommended_sprint": "<Sprint 1|2|3|4|5|6>",
        "engineer_level_required": "junior|mid|senior",
        "ai_automation_potential": "high|medium|low",
        "ai_automation_notes": "<what AI can/cannot do for this service>"
      }
    ],
    "summary": {
      "green_count": <int>,
      "amber_count": <int>,
      "red_count": <int>,
      "recommended_total_sprints": <int>,
      "highest_risk_service": "<service_id>",
      "quick_wins": ["<service_id>"]
    }
  }

  SERVICE MANIFEST (from 1.1-A):
  <<<PASTE 1.1-A OUTPUT HERE>>>

  DEPENDENCY GRAPH (from 1.2-A):
  <<<PASTE 1.2-A OUTPUT HERE>>>

  WSDL REGISTRY (from 1.3-C):
  <<<PASTE 1.3-C OUTPUT HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 1.4-B  ❯  Migration Sequencing Plan
#               (run after 1.4-A to produce sprint plan)
# ─────────────────────────────────────────────────────────────

PROMPT_1_4_B: |
  You are a senior delivery manager with Java migration
  experience.

  Using the risk classification below, produce a sprint
  sequencing plan for the full migration.

  CONSTRAINTS:
  - Team size: [TEAM_SIZE] engineers
  - Sprint length: [SPRINT_LENGTH_WEEKS] weeks
  - Max parallel services per sprint: [MAX_PARALLEL]
  - Red tier services must have at least one senior engineer
    assigned
  - No service with internal_service_calls should be
    migrated before the services it depends on
  - The common-lib and Helm library chart (Phase 2) must
    be completed before Sprint 1 of Phase 3

  OUTPUT SCHEMA:
  {
    "plan_metadata": {
      "team_size": <int>,
      "sprint_length_weeks": <int>,
      "total_sprints": <int>,
      "estimated_total_weeks": <int>
    },
    "phases": [
      {
        "phase": "Phase 2 - Pilot",
        "sprints": [
          {
            "sprint_number": 0,
            "focus": "Common lib + Helm library + Harness template",
            "services": [],
            "engineer_allocation": "<description>",
            "ai_tasks": ["<what Copilot does this sprint>"],
            "human_tasks": ["<what engineers must do manually>"],
            "exit_criteria": ["<done means...>"]
          }
        ]
      },
      {
        "phase": "Phase 3 - Full Migration",
        "sprints": [
          {
            "sprint_number": <int>,
            "services": [
              {
                "service_id": "<string>",
                "tier": "Green|Amber|Red",
                "assigned_engineer_level": "junior|mid|senior"
              }
            ],
            "ai_tasks": ["<Copilot tasks>"],
            "human_tasks": ["<engineer tasks>"],
            "dependencies_met": <bool>,
            "exit_criteria": ["<done means...>"]
          }
        ]
      }
    ],
    "decommission_schedule": [
      {
        "service_id": "<string>",
        "earliest_decommission_sprint": <int>,
        "notes": "<strangler fig cutover notes>"
      }
    ]
  }

  RISK CLASSIFICATION (from 1.4-A):
  <<<PASTE 1.4-A OUTPUT HERE>>>


# ============================================================
# PHASE 1 — EXECUTION ORDER
# ============================================================
#
# Run prompts in this exact sequence to avoid missing inputs:
#
#  1. PROMPT_1_1_A  → produces: migration manifest JSON
#                               (re-run 1.1-B for low_confidence services)
#
#  2. PROMPT_1_2_A  → input:  manifest from 1.1-A
#                     produces: dependency graph JSON
#                               (re-run 1.2-B if internal calls missing)
#
#  3. PROMPT_1_3_A  → run once per WSDL file
#                     produces: individual WSDL audit reports
#
#  4. PROMPT_1_3_B  → run once per WSDL/Reference.cs pair
#                     produces: drift detection report
#
#  5. PROMPT_1_3_C  → input:  all 1.3-A reports
#                     produces: WSDL registry JSON
#
#  6. PROMPT_1_4_A  → input:  1.1-A + 1.2-A + 1.3-C
#                     produces: risk classification JSON
#
#  7. PROMPT_1_4_B  → input:  1.4-A + team constraints
#                     produces: sprint sequencing plan
#
# All JSON outputs should be committed to:
#   /docs/migration/phase1/
#     manifest.json          (1.1-A)
#     dependency-graph.json  (1.2-A)
#     wsdl-registry.json     (1.3-C)
#     risk-classification.json (1.4-A)
#     sprint-plan.json       (1.4-B)
#
# These five files collectively ARE the Phase 1 deliverable.
# ============================================================
