# Migration Copilot Instructions
# .NET WCF → Java Spring Boot — Phase 2: Application Migration
#
# SCOPE: Pure application code only.
#        No Docker, no Helm, no AKS, no Harness.
#        Those belong to Phase 3 (DevOps & Infrastructure).
#
# USAGE:
#   GitHub Copilot Chat : prefix with @workspace when running
#                         inside the IDE against the legacy
#                         .NET solution or the new Java repo
#   Standalone LLM      : paste the prompt block directly;
#                         attach or paste source files as shown
#
# PREREQUISITE INPUTS (all produced by Phase 1):
#   manifest.json          → service inventory
#   wsdl-registry.json     → SAP WSDL audit results
#   risk-classification.json → tier scores per service
#
# EXECUTION ORDER:
#   2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 2.6 → 2.7
#   Each activity's output feeds the next.
#   Run 2.1 once per WSDL. Run 2.2–2.6 once per service.
#   Run 2.7 once at the end to assemble the common-lib.
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
# 1.  Target language is Java 21. Use records for immutable
#     DTOs, sealed interfaces for closed type hierarchies,
#     switch expressions over switch statements.
#
# 2.  Target framework is Spring Boot 4 with Spring Web MVC
#     and Spring WS. No reactive (WebFlux) unless explicitly
#     flagged in the manifest.
#
# 3.  All SAP SOAP calls go through WebServiceTemplate.
#     Never use raw JAX-WS DispatchClient or HttpURLConnection.
#
# 4.  All decimal/monetary values MUST use BigDecimal.
#     Never float or double for financial fields.
#
# 5.  Preserve every business rule found in the .NET source.
#     If a rule's intent is unclear, add a
#     // TODO: CONFIRM BUSINESS RULE — <description>
#     comment rather than silently dropping it.
#
# 6.  Never invent values. If a mapping cannot be determined,
#     set it to null and add review_flag: true in the JSON
#     output, or a TODO comment in the code output.
#
# 7.  Flag all financially sensitive code with:
#     // FINANCIAL: <reason> — requires senior engineer review
#
# 8.  Generated code must compile. If a dependency or import
#     cannot be resolved, note it as:
#     // IMPORT NEEDED: <fully qualified class>
#
# 9.  All method and class names use camelCase / PascalCase
#     Java conventions. Do not carry over C# PascalCase method
#     names (e.g. ProcessPayment → processPayment).
#
# 10. Return only the requested code or JSON. No preamble,
#     no explanation prose, no markdown code fences unless
#     the prompt explicitly asks for explanation.


# ============================================================
# ACTIVITY 2.1 — WSDL TO JAVA STUB GENERATION
# ============================================================
#
# PURPOSE:
#   Configure jaxb2-maven-plugin to generate Java source
#   from each SAP WSDL. This directly replaces svcutil.exe
#   and the auto-generated Reference.cs files.
#
# WHEN TO USE:
#   Run once per WSDL entry in wsdl-registry.json.
#   Re-run if the SAP team updates the WSDL.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.1-A  ❯  Generate pom.xml plugin config for one WSDL
# ─────────────────────────────────────────────────────────────

PROMPT_2_1_A: |
  You are a senior Java Spring Boot engineer.

  Generate the jaxb2-maven-plugin <execution> block
  for the WSDL described below.

  RULES:
  - Use jaxb2-maven-plugin version 3.x
  - Source path: src/main/resources/wsdl/<wsdl_name>
  - Target package: com.[ORG].[SERVICE_NAME].sap.generated
  - If the WSDL audit (from Phase 1) flagged any
    non-standard SAP extensions, add the appropriate
    xjc binding customisation file reference
  - If any XSD type maps to BigDecimal (monetary/decimal
    fields), generate a bindings.xjb entry enforcing that
  - Add a comment above each execution block identifying
    which SAP service it serves

  OUTPUT: A ready-to-paste <execution> XML block plus any
  required bindings.xjb content. No surrounding prose.

  WSDL AUDIT ENTRY (from wsdl-registry.json):
  <<<PASTE SINGLE WSDL ENTRY FROM wsdl-registry.json>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.1-B  ❯  Validate generated stub classes
#               (run after mvn generate-sources)
# ─────────────────────────────────────────────────────────────

PROMPT_2_1_B: |
  You are a senior Java Spring Boot engineer.

  Review the JAXB-generated Java stubs below against the
  original WSDL audit and the legacy Reference.cs.

  CHECK FOR:
  1. Every operation in the WSDL has a corresponding
     request/response class in the generated output
  2. All decimal fields are BigDecimal (not double/float)
  3. All date/time fields are java.time types
     (not java.util.Date or XMLGregorianCalendar)
  4. Field names are idiomatic Java camelCase
     (WSDL may use PascalCase or snake_case)
  5. Any field present in Reference.cs but missing from
     the generated stubs — flag as drift risk
  6. Any SAP fault type has a corresponding generated class

  OUTPUT SCHEMA:
  {
    "validation_passed": <bool>,
    "issues": [
      {
        "severity": "blocker|warning|info",
        "field_or_class": "<name>",
        "issue": "<description>",
        "fix": "<recommended action>"
      }
    ],
    "drift_from_reference_cs": [
      {
        "item": "<field or class>",
        "in_reference_cs": true,
        "in_generated": false,
        "risk": "<migration impact>"
      }
    ],
    "manual_binding_adjustments_needed": ["<description>"]
  }

  WSDL AUDIT (from Phase 1 1.3-A):
  <<<PASTE WSDL AUDIT JSON>>>

  REFERENCE.CS EXCERPT (relevant types only):
  <<<PASTE C# TYPES FROM Reference.cs>>>

  GENERATED JAVA STUBS (file listing):
  <<<PASTE GENERATED JAVA CLASS NAMES AND KEY FIELDS>>>


# ============================================================
# ACTIVITY 2.2 — DTO TRANSLATION
# ============================================================
#
# PURPOSE:
#   Translate every [DataContract] C# class used internally
#   (i.e. not SAP-generated types) into idiomatic Java 21
#   records or classes.
#
# NOTE:
#   SAP types are already handled by 2.1 (JAXB generation).
#   This activity covers only the service's OWN DTOs —
#   the request/response objects exposed to callers.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.2-A  ❯  Translate a single [DataContract] class
# ─────────────────────────────────────────────────────────────

PROMPT_2_2_A: |
  You are a senior Java 21 engineer.

  Translate the C# [DataContract] class below into a
  Java 21 record.

  RULES:
  - Use Java 21 record syntax
  - All monetary/decimal fields → BigDecimal
  - All string fields → String (nullable via @Nullable)
  - All date fields → LocalDate or LocalDateTime
    (pick based on whether time component is present)
  - [DataMember(IsRequired = true)] → add
    // REQUIRED field comment
  - Preserve all field names in camelCase
  - If the class has validation annotations in C#
    (e.g. [Required], [Range], [StringLength]), add
    equivalent Jakarta Bean Validation annotations
    (@NotNull, @NotBlank, @DecimalMin, @Size)
  - If the class has a default value in C# (e.g. = "GBP"),
    implement it via a compact constructor in the record
  - Add package declaration:
    package com.[ORG].[SERVICE_NAME].dto;
  - Add import statements for all referenced types

  OUTPUT: Complete Java record source file. Code only.

  C# DataContract SOURCE:
  <<<PASTE C# DataContract CLASS HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.2-B  ❯  Bulk DTO translation for a full service
#               (use when a service has 5+ DataContracts)
# ─────────────────────────────────────────────────────────────

PROMPT_2_2_B: |
  You are a senior Java 21 engineer.

  Translate ALL [DataContract] classes in the source below
  into Java 21 records. Apply all rules from PROMPT_2_2_A
  to each class.

  Additionally:
  - If multiple DTOs share common fields (e.g. requestId,
    correlationId, timestamp), extract those into a shared
    base record using record composition or an interface
  - Flag any enum types ([DataMember] with a fixed set of
    string values) — these should become Java enums, not
    Strings. List them under "enum_candidates" in a
    // TODO: CONVERT TO ENUM comment in the output

  OUTPUT: One Java record file per DataContract class,
  clearly delimited with:
  // ===== FILE: [ClassName].java =====

  C# SOURCE (all DataContract classes for this service):
  <<<PASTE C# DataContract CLASSES HERE>>>


# ============================================================
# ACTIVITY 2.3 — BUSINESS LOGIC TRANSLATION
# ============================================================
#
# PURPOSE:
#   Translate the core service implementation (.svc.cs) into
#   a Spring Boot @Service class and @RestController endpoint.
#   This is the highest-value and highest-risk activity.
#
# APPROACH:
#   Run 2.3-A first (pre-translation analysis).
#   Then 2.3-B (generate the @Service).
#   Then 2.3-C (generate the @RestController).
#   Engineer reviews all three outputs before committing.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.3-A  ❯  Pre-translation business logic analysis
#               (run BEFORE generating any Java code)
# ─────────────────────────────────────────────────────────────

PROMPT_2_3_A: |
  You are a senior Java migration engineer.

  Analyse the C# WCF service implementation below WITHOUT
  generating any Java code yet.

  EXTRACT AND DOCUMENT:

  1. Pre-SAP logic (everything before the SAP client call):
     - Input validation rules (what makes a request invalid?)
     - Data enrichment (fields defaulted, derived, or looked up)
     - Business rule checks (domain-specific conditions)

  2. SAP call details:
     - Which SAP operation is called
     - How internal fields map TO SAP request fields
     - Any fields constructed rather than passed through

  3. Post-SAP logic (everything after the SAP response):
     - How SAP response fields map TO internal response fields
     - Any response enrichment or transformation
     - Any audit logging or side effects

  4. Error handling:
     - Every FaultException catch block and what it does
     - Any non-SAP exceptions caught (null ref, format, etc.)
     - Any error codes produced

  5. Hidden complexity flags:
     - Hardcoded values (magic strings, default currencies)
     - Calls to shared utilities (what do they do?)
     - Any conditional branching based on business state
     - Financial calculations (flag with FINANCIAL: tag)
     - Any calls to other internal WCF services

  OUTPUT SCHEMA:
  {
    "service_name": "<string>",
    "operation_name": "<string>",
    "pre_sap_logic": [
      { "step": "<description>", "type": "validation|enrichment|rule_check", "financial": <bool> }
    ],
    "sap_call": {
      "operation": "<SAP operation name>",
      "field_mappings_to_sap": [
        { "internal_field": "<string>", "sap_field": "<string>", "transform": "<none|default|derived|constructed>" }
      ]
    },
    "post_sap_logic": [
      { "step": "<description>", "type": "mapping|enrichment|audit|side_effect", "financial": <bool> }
    ],
    "error_handling": [
      { "exception_type": "<string>", "error_code_produced": "<string>", "action": "<description>" }
    ],
    "hidden_complexity": [
      { "type": "hardcoded_value|shared_util|conditional_branch|financial_calc|internal_service_call", "description": "<string>", "migration_action": "<string>" }
    ],
    "migration_risk": "low|medium|high",
    "migration_risk_reason": "<string>"
  }

  C# SERVICE IMPLEMENTATION (.svc.cs):
  <<<PASTE C# IMPLEMENTATION HERE>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.3-B  ❯  Generate the Spring Boot @Service class
# ─────────────────────────────────────────────────────────────

PROMPT_2_3_B: |
  You are a senior Java 21 Spring Boot engineer.

  Using the pre-translation analysis below (from 2.3-A)
  and the C# source, generate the Spring Boot @Service
  class for this service.

  RULES:
  - Class name: [ServiceName]Service
  - Package: com.[ORG].[SERVICE_NAME].service
  - Inject the SAP client via constructor injection
  - Every pre-SAP validation rule must be preserved exactly
  - All field mappings to SAP must match the analysis
  - All field mappings from SAP response must match
  - All FaultException catch blocks must become catches
    on SoapFaultClientException with equivalent error codes
  - Hardcoded values must be externalised to:
    @Value("${[service].[property]}") fields
    and noted in a // CONFIG NEEDED: application.yml comment
  - Shared utility calls must reference the migrated Java
    utility (if not yet migrated, add a TODO comment)
  - All financial calculations must have a
    // FINANCIAL: <description> comment
  - Use SLF4J Logger (not System.out.println)
  - Use structured log parameters: log.info("msg {}", val)
    not log.info("msg " + val)

  OUTPUT: Complete @Service Java source file. Code only.

  PRE-TRANSLATION ANALYSIS (from 2.3-A):
  <<<PASTE 2.3-A JSON OUTPUT>>>

  C# ORIGINAL SOURCE:
  <<<PASTE .svc.cs CONTENT>>>

  SAP GENERATED STUBS PACKAGE:
  com.[ORG].[SERVICE_NAME].sap.generated

# ─────────────────────────────────────────────────────────────
# PROMPT 2.3-C  ❯  Generate the Spring Boot @RestController
# ─────────────────────────────────────────────────────────────

PROMPT_2_3_C: |
  You are a senior Java 21 Spring Boot engineer.

  Generate the @RestController endpoint for the service
  described below.

  RULES:
  - Class name: [ServiceName]Controller
  - Package: com.[ORG].[SERVICE_NAME].controller
  - Base path: /api/v1/[resource-name-plural-kebab]
  - Use @PostMapping for operations that mutate SAP data
  - Use @GetMapping for operations that only read SAP data
  - Inject @Service class via constructor injection
  - Use @RequestBody @Valid for all POST request bodies
  - Return ResponseEntity<T> for all methods
  - Add @ExceptionHandler methods for:
      - [ServiceName]Exception → 400 Bad Request with
        { "code": "...", "message": "..." }
      - SoapFaultClientException → 502 Bad Gateway with
        { "code": "SAP_FAULT", "message": "..." }
      - Generic Exception → 500 Internal Server Error
        (log full stack trace, return sanitised message)
  - Add @Operation annotation stubs (SpringDoc OpenAPI)
    for each endpoint — leave summary/description as TODO
  - DO NOT include any business logic in the controller —
    it belongs exclusively in the @Service class

  OUTPUT: Complete @RestController Java source file.
  Code only.

  SERVICE CLASS NAME AND METHODS (from 2.3-B output):
  <<<PASTE @Service CLASS SIGNATURE AND METHOD SIGNATURES>>>

  DTO CLASSES (from 2.2 output):
  <<<PASTE DTO RECORD NAMES AND FIELDS>>>


# ============================================================
# ACTIVITY 2.4 — SAP CLIENT LAYER
# ============================================================
#
# PURPOSE:
#   Generate the WebServiceTemplate-based SAP client class
#   and its Spring @Configuration wiring. This directly
#   replaces ClientBase<T> and web.config bindings.
#
# NOTE:
#   One client class per SAP WSDL / service group.
#   One @Configuration class covers all clients that share
#   the same SAP host and security scheme (use the
#   shared_bean_groups from wsdl-registry.json to decide).
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.4-A  ❯  Generate the SAP client @Component
# ─────────────────────────────────────────────────────────────

PROMPT_2_4_A: |
  You are a senior Java Spring Boot engineer specialising
  in Spring-WS and SAP SOAP integration.

  Generate a Spring @Component SAP client class for the
  WSDL described below.

  RULES:
  - Class name: Sap[ServiceName]Client
  - Package: com.[ORG].[SERVICE_NAME].client
  - Inject WebServiceTemplate via constructor
  - One public method per SAP operation in the WSDL
  - Each method signature:
      public [ResponseType] [operationName]([RequestType] request)
  - Use webServiceTemplate.marshalSendAndReceive(uri, request)
  - URI comes from @Value("${sap.[service].endpoint}")
  - Catch SoapFaultClientException at this layer only for
    logging — rethrow for the @Service to handle
  - Add SLF4J structured logging on entry and exit:
      log.debug("SAP [Operation] request: account={}", ...)
      log.debug("SAP [Operation] response: docNum={}", ...)
  - Log at WARN level if SAP response status is not SUCCESS
  - Never log full request/response bodies at INFO level
    (they may contain sensitive financial data)
  - Add a // TIMEOUT: confirm with SAP team comment
    for any operation flagged medium/high complexity
    in the WSDL audit

  OUTPUT: Complete @Component Java source file. Code only.

  WSDL AUDIT ENTRY (from Phase 1 1.3-A):
  <<<PASTE WSDL AUDIT JSON>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.4-B  ❯  Generate the SAP @Configuration bean wiring
# ─────────────────────────────────────────────────────────────

PROMPT_2_4_B: |
  You are a senior Java Spring Boot engineer.

  Generate the Spring @Configuration class that wires
  WebServiceTemplate, Jaxb2Marshaller, and
  Wss4jSecurityInterceptor for the SAP connection group
  described below.

  RULES:
  - Class name: Sap[GroupName]WebServiceConfig
  - Package: com.[ORG].config
  - Read credentials from:
      @Value("${sap.[group].username}")
      @Value("${sap.[group].password}")
  - Read endpoint from:
      @Value("${sap.[group].endpoint}")
  - Jaxb2Marshaller must scan ALL generated stub packages
    for this connection group
  - WS-Security setup based on the security_scheme field
    in the shared_bean_group entry:
      UsernameToken → Wss4jSecurityInterceptor with
        securementActions = "UsernameToken"
        securementPasswordType = WSConstants.PW_TEXT
      Certificate   → add // TODO: CERT CONFIG NEEDED
                        with Key Vault reference comment
      None          → no interceptor, add a warning comment
  - Add HttpComponentsMessageSender with timeouts:
      connectTimeout from ${sap.[group].connect-timeout-ms:5000}
      readTimeout    from ${sap.[group].read-timeout-ms:60000}
  - Add a companion application.yml snippet as a block
    comment at the bottom of the file showing all required
    property keys for this config class

  OUTPUT: Complete @Configuration Java source file
  followed by the application.yml snippet. Code only.

  SHARED BEAN GROUP ENTRY (from wsdl-registry.json):
  <<<PASTE shared_bean_groups ENTRY>>>


# ============================================================
# ACTIVITY 2.5 — EXCEPTION MAPPING
# ============================================================
#
# PURPOSE:
#   Create a consistent, service-wide exception hierarchy
#   that maps C# FaultException<T> types and SAP SOAP faults
#   to Java RuntimeExceptions. One hierarchy per service.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.5-A  ❯  Generate exception hierarchy for a service
# ─────────────────────────────────────────────────────────────

PROMPT_2_5_A: |
  You are a senior Java 21 engineer.

  Generate a complete exception hierarchy for the service
  described below, based on the FaultException types found
  in the C# source.

  RULES:
  - Root exception: [ServiceName]Exception extends RuntimeException
    - Fields: String code, String message
    - Constructor: (String code, String message)
    - Constructor: (String code, String message, Throwable cause)

  - One subclass per distinct FaultContract type in the C# source:
    e.g. [ServiceName]ValidationException extends [ServiceName]Exception
         [ServiceName]SapException extends [ServiceName]Exception

  - A static factory class [ServiceName]Exceptions with named
    factory methods for every error code found in the C# source:
    e.g. [ServiceName]Exceptions.invalidAccount()
         [ServiceName]Exceptions.invalidAmount()
         [ServiceName]Exceptions.sapFault(String sapMessage)

  - Package: com.[ORG].[SERVICE_NAME].exception

  - For each error code factory method, add a
    // SAP_ERROR_CODE: [code] comment if the error maps
    directly to a SAP fault error code (from WSDL audit)

  OUTPUT: One Java file per class, delimited with:
  // ===== FILE: [ClassName].java =====
  Code only.

  C# SERVICE SOURCE (focus on FaultException usage):
  <<<PASTE .svc.cs CONTENT>>>

  WSDL FAULT TYPES (from Phase 1 1.3-A operations[].fault_messages):
  <<<PASTE FAULT MESSAGE NAMES AND FIELDS>>>


# ============================================================
# ACTIVITY 2.6 — UNIT AND CONTRACT TESTS
# ============================================================
#
# PURPOSE:
#   Generate unit tests for the @Service class and
#   Spring Cloud Contract stubs for the REST endpoint.
#   These become the permanent regression suite for the
#   Strangler Fig cutover in Phase 4.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.6-A  ❯  Generate @Service unit tests
# ─────────────────────────────────────────────────────────────

PROMPT_2_6_A: |
  You are a senior Java Spring Boot engineer.

  Generate JUnit 5 unit tests for the @Service class below.

  RULES:
  - Test class name: [ServiceName]ServiceTest
  - Package: com.[ORG].[SERVICE_NAME].service
  - Use Mockito to mock the SAP client (@ExtendWith(MockitoExtension.class))
  - One test method per business rule identified in the
    pre-translation analysis (2.3-A output)
  - Test naming convention:
      should_[expectedBehaviour]_when_[condition]
  - Cover ALL validation rules — each invalid input
    must have its own test asserting the correct
    exception type and error code
  - Cover the happy path with a realistic SAP response
  - Cover SAP fault response (SoapFaultClientException)
    and assert it maps to the correct service exception
  - For financial calculations: add boundary tests
    (zero, negative, maximum expected value)
  - Use AssertJ assertions (not JUnit 5 assertEquals)
  - Use @DisplayName for human-readable test descriptions

  OUTPUT: Complete test class Java source. Code only.

  @SERVICE SOURCE (from 2.3-B):
  <<<PASTE @Service JAVA SOURCE>>>

  PRE-TRANSLATION ANALYSIS (from 2.3-A):
  <<<PASTE 2.3-A JSON — focuses on validation rules>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.6-B  ❯  Generate Spring Cloud Contract stubs
# ─────────────────────────────────────────────────────────────

PROMPT_2_6_B: |
  You are a senior Java Spring Boot engineer specialising
  in consumer-driven contract testing.

  Generate Spring Cloud Contract .groovy contract files
  for the REST endpoint described below.

  RULES:
  - One contract file per scenario (happy path + each
    error case)
  - File naming: [operation]_[scenario].groovy
    e.g. process_payment_success.groovy
         process_payment_invalid_account.groovy
         process_payment_sap_fault.groovy
  - Use Contract.make { } DSL syntax
  - Request bodies must use realistic but anonymised data
    (no real account numbers, no real amounts > 1000.00)
  - Response bodies must use matchers (anyNonBlankString(),
    anyPositiveInt()) not hardcoded values where the real
    system would produce variable output
  - For error responses, assert the exact error code
    (these are fixed and must not drift)
  - Add a description field to every contract explaining
    the business scenario being tested
  - Place contracts in:
    src/test/resources/contracts/[service-name]/

  OUTPUT: One Groovy contract file per scenario, delimited:
  // ===== FILE: [filename].groovy =====
  Code only.

  @RestController ENDPOINTS (from 2.3-C):
  <<<PASTE CONTROLLER METHOD SIGNATURES AND MAPPINGS>>>

  DTO RECORDS (from 2.2):
  <<<PASTE REQUEST AND RESPONSE RECORD DEFINITIONS>>>

  EXCEPTION CODES (from 2.5-A factories):
  <<<PASTE FACTORY METHOD NAMES AND ERROR CODES>>>


# ============================================================
# ACTIVITY 2.7 — COMMON-LIB ASSEMBLY
# ============================================================
#
# PURPOSE:
#   Identify all classes that are genuinely reusable across
#   multiple migrated services and consolidate them into a
#   shared common-lib Maven module.
#   Run this ONCE after the first 2-3 services are migrated.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.7-A  ❯  Identify common-lib candidates
# ─────────────────────────────────────────────────────────────

PROMPT_2_7_A: |
  You are a senior Java architect.

  Review the migrated service classes listed below and
  identify which components belong in a shared common-lib
  module rather than in individual service modules.

  CANDIDATE CRITERIA:
  - Used by 2 or more services (exact same logic)
  - Infrastructure / cross-cutting (logging, correlation IDs,
    error response formatting, SAP WS config)
  - No service-specific business logic

  DO NOT include in common-lib:
  - Service-specific exception types
  - Service-specific DTOs
  - Service-specific SAP client classes
  - Any class containing business rules

  FOR EACH CANDIDATE OUTPUT:
  {
    "class_name": "<string>",
    "current_location": "<service module>",
    "used_by_services": ["<service_id>"],
    "proposed_package": "com.[ORG].common.<subpackage>",
    "generalisation_needed": "<any changes to make it truly generic>",
    "risk": "safe_to_move|needs_refactor|keep_in_service"
  }

  THEN generate:
  1. The common-lib pom.xml (parent module, no service deps)
  2. The proposed package structure as a directory tree
  3. For each "safe_to_move" class: the generalised Java
     source with package updated and service-specific
     references removed

  MIGRATED SERVICE CLASSES (list class names + packages):
  <<<PASTE CLASS NAMES FROM ALL MIGRATED SERVICES SO FAR>>>

  SHARED UTILITIES FROM .NET (from Phase 1 manifest.json
  shared_utilities array):
  <<<PASTE shared_utilities ARRAY>>>

# ─────────────────────────────────────────────────────────────
# PROMPT 2.7-B  ❯  Generate the common-lib CorrelationId filter
#               (always needed — generate once early)
# ─────────────────────────────────────────────────────────────

PROMPT_2_7_B: |
  You are a senior Java Spring Boot engineer.

  Generate a servlet filter that handles correlation ID
  propagation for all migrated Spring Boot services.
  This replaces any request-tracing utilities in the
  .NET codebase and integrates with Istio's trace headers.

  REQUIREMENTS:
  - Class: CorrelationIdFilter implements OncePerRequestFilter
  - Package: com.[ORG].common.filter
  - On every inbound request:
      Read X-Correlation-Id header if present
      Generate a UUID if header is absent
      Store in MDC key "correlationId"
      Add to response header X-Correlation-Id
  - Also propagate Istio trace headers if present:
      x-request-id, x-b3-traceid, x-b3-spanid
      (store in MDC for log correlation)
  - Register as a @Bean in a CommonFilterConfig @Configuration
  - Add an application.yml snippet showing how to configure
    filter order

  OUTPUT: Two Java files (filter + config) delimited with:
  // ===== FILE: [ClassName].java =====
  Code only.


# ============================================================
# PHASE 2 — CODE REVIEW CHECKLIST
# ============================================================
#
# Run this prompt on every generated service BEFORE merging.
#
# ─────────────────────────────────────────────────────────────
# PROMPT 2.R  ❯  Pre-merge code review
# ─────────────────────────────────────────────────────────────

PROMPT_2_R: |
  You are a senior Java Spring Boot code reviewer with
  experience in financial systems and SAP integration.

  Review the migrated service code below against the
  original C# source and the pre-translation analysis.

  CHECK EACH ITEM AND MARK: PASS | FAIL | WARNING

  CORRECTNESS:
  [ ] Every validation rule in 2.3-A pre_sap_logic is present
  [ ] Every SAP field mapping matches 2.3-A sap_call.field_mappings_to_sap
  [ ] Every error code matches the original FaultException codes
  [ ] All decimal fields use BigDecimal (no float/double)
  [ ] All date fields use java.time types

  SAFETY:
  [ ] No hardcoded credentials or endpoint URLs in code
  [ ] No financial data logged at INFO or DEBUG level
  [ ] All @Value properties have sensible defaults or
      are clearly required (no silent null)
  [ ] SoapFaultClientException is caught and rethrown as
      service exception (never swallowed)

  QUALITY:
  [ ] No business logic in @RestController
  [ ] All TODO comments have a clear action and owner
  [ ] All FINANCIAL: comments have been acknowledged
  [ ] Logger uses parameterised format, not concatenation
  [ ] Constructor injection used throughout (no @Autowired field injection)

  TESTS:
  [ ] Every validation rule has a unit test
  [ ] Happy path has a unit test
  [ ] SAP fault path has a unit test
  [ ] Contract stubs cover happy path + all error codes

  OUTPUT SCHEMA:
  {
    "service_id": "<string>",
    "review_passed": <bool>,
    "items": [
      {
        "category": "correctness|safety|quality|tests",
        "check": "<item description>",
        "status": "PASS|FAIL|WARNING",
        "finding": "<description if not PASS>",
        "action_required": "<string or null>"
      }
    ],
    "merge_recommendation": "approve|approve_with_actions|block",
    "block_reasons": ["<string>"]
  }

  MIGRATED JAVA SOURCE (all files for this service):
  <<<PASTE ALL GENERATED JAVA FILES>>>

  ORIGINAL C# SOURCE:
  <<<PASTE .svc.cs>>>

  PRE-TRANSLATION ANALYSIS (from 2.3-A):
  <<<PASTE 2.3-A JSON>>>


# ============================================================
# PHASE 2 — EXECUTION ORDER SUMMARY
# ============================================================
#
# Per WSDL (run once per entry in wsdl-registry.json):
#   2.1-A → pom.xml plugin config
#   mvn generate-sources  (manual step)
#   2.1-B → validate generated stubs
#
# Per service (run for each service in manifest.json,
#              ordered by risk tier — Green first):
#   2.2-A or 2.2-B → translate DTOs
#   2.3-A          → pre-translation analysis  ← CRITICAL
#   2.3-B          → generate @Service
#   2.3-C          → generate @RestController
#   2.4-A          → generate SAP client
#   2.4-B          → generate SAP @Configuration (once per group)
#   2.5-A          → generate exception hierarchy
#   2.6-A          → generate @Service unit tests
#   2.6-B          → generate contract stubs
#   2.R            → pre-merge review
#
# After first 2-3 services are complete:
#   2.7-A → identify common-lib candidates
#   2.7-B → generate CorrelationId filter (if not yet done)
#
# Commit outputs to:
#   [service-name]/src/main/java/...  (service code)
#   [service-name]/src/test/java/...  (unit tests)
#   [service-name]/src/test/resources/contracts/...  (contracts)
#   common-lib/src/main/java/...  (shared utilities)
#
# DO NOT proceed to Phase 3 (DevOps) until:
#   ✓ At least 2 Green-tier services pass PROMPT_2_R review
#   ✓ All contract tests pass
#   ✓ common-lib module builds cleanly
#   ✓ All FINANCIAL: comments have senior engineer sign-off
# ============================================================
