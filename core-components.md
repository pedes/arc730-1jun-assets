MuleSoft Core Components — Use Cases & Best Practices

  ---
  Batch
  
  Batch Job + Batch Step + Batch Aggregator

  Best for: Large-volume, record-by-record processing where you can tolerate partial failures.

  ┌─────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────────┐
  │                Use Case                 │                                            How                                            │
  ├─────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────┤
  │ Nightly ETL — load 500k rows from a DB  │ Batch Job splits the result set into chunks; each Batch Step upserts records; Batch       │
  │ into Salesforce                         │ Aggregator commits every 100 records                                                      │
  ├─────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────┤
  │ CSV file ingestion                      │ Batch Job reads the file; one Step validates each row; a second Step writes valid rows to │
  │                                         │  DB; Aggregator flushes invalid rows to an error queue                                    │
  ├─────────────────────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────────┤
  │ Mass email/notification dispatch        │ Each record = one recipient; Batch Step calls the email API; failures are isolated per    │
  │                                         │ record and reported in the On Complete phase                                              │
  └─────────────────────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────────┘

  Key rule: Use Batch when a single failure in one record should not abort the entire job. For small collections (< a few hundred items),
  For Each is simpler and sufficient.

  ---
  Components
  
  Logger

  Best for: Observability — trace flow execution, capture variable state, record errors.

  <logger level="INFO"
          message="#['Order received — ID: ' ++ vars.orderId ++ ' | correlationId: ' ++ correlationId]"/>

  ┌─────────────────────────────────────────────────────────────────────────┬───────┐
  │                                Use Case                                 │ Level │
  ├─────────────────────────────────────────────────────────────────────────┼───────┤
  │ Trace entry/exit of a flow                                              │ DEBUG │
  ├─────────────────────────────────────────────────────────────────────────┼───────┤
  │ Record business-relevant events (order placed, payment approved)        │ INFO  │
  ├─────────────────────────────────────────────────────────────────────────┼───────┤
  │ Warn on degraded-but-recoverable state (fallback used, retry attempted) │ WARN  │
  ├─────────────────────────────────────────────────────────────────────────┼───────┤
  │ Record caught exceptions before re-throwing                             │ ERROR │
  └─────────────────────────────────────────────────────────────────────────┴───────┘

  Best practice: Always include correlationId in the message pattern. It threads together all log lines for a single request across flows.

  ---
  Transform Message (DataWeave)
  
  Best for: Any structural or format conversion — JSON↔XML, flattening, enrichment, filtering.

  ┌───────────────────────────────────────────────────────┬────────────────────────────────────────────────────────────┐
  │                       Use Case                        │                          Example                           │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Map upstream API response to internal canonical model │ Rename fields, change date formats, flatten nested objects │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Filter a collection before passing downstream         │ payload filter $.status == "ACTIVE"                        │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Build a request body for an outbound call             │ Construct JSON from multiple variables                     │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Convert XML SOAP response to JSON REST response       │ Full format translation                                    │
  └───────────────────────────────────────────────────────┴────────────────────────────────────────────────────────────┘

  Best practice: Keep all transformation logic in DataWeave scripts, not in MEL expressions or Set Payload strings. Scripts are testable and
  version-controllable.

  ---
  Flow Reference
  
  Best for: Decomposing a large flow into reusable, single-responsibility subflows.

  main-flow
    ├── flowRef → validate-input-subflow
    ├── flowRef → enrich-with-customer-subflow
    └── flowRef → persist-order-subflow

  ┌───────────────────────────────────────────────────────┬────────────────────────────────────────────────────────────┐
  │                       Use Case                        │                            How                             │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Shared validation logic reused across multiple flows  │ Extract to a subflow; reference from each entry-point flow │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Breaking a long flow into readable named steps        │ Each subflow = one business step                           │
  ├───────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┤
  │ Calling a private flow that has its own error handler │ Use a named <flow> (not <sub-flow>) as the target          │
  └───────────────────────────────────────────────────────┴────────────────────────────────────────────────────────────┘

  Best practice: Subflows inherit the error handling of their caller. If you need isolated error handling for a section, use a <flow> target
  (with its own <error-handler>) or wrap the flowRef in a Try scope.

  ---
  Dynamic Evaluate
  
  Best for: When the transformation script itself is data-driven — selected at runtime based on message content.

  ┌────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────┐
  │                            Use Case                            │                                 How                                 │
  ├────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ Multi-tenant transformations — each client has its own mapping │ Store scripts in a DB or Object Store keyed by clientId; load and   │
  │  rules                                                         │ evaluate at runtime                                                 │
  ├────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ A/B testing two DataWeave scripts in production                │ Select script by feature flag in a variable                         │
  ├────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
  │ Rule-engine lite — route different document types to different │ Derive script name from payload.documentType                        │
  │  transform scripts                                             │                                                                     │
  └────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────┘

  Best practice: Cache the script content in a variable using the Cache scope or Object Store to avoid re-fetching on every message.

  ---
  Parse Template
  
  Best for: Generating text output (emails, HTML, SQL, XML payloads) from a template file + dynamic values.

  ┌────────────────────────────────────────────────────┬──────────────────────────────────────────────────────┐
  │                      Use Case                      │                   Template content                   │
  ├────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ Notification email body                            │ HTML with #[vars.customerName] placeholders          │
  ├────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ Generating a dynamic SQL file (non-connector path) │ SELECT * FROM orders WHERE date = #[vars.reportDate] │
  ├────────────────────────────────────────────────────┼──────────────────────────────────────────────────────┤
  │ Building an XML payload for a legacy SOAP API      │ Static envelope with injected values                 │
  └────────────────────────────────────────────────────┴──────────────────────────────────────────────────────┘

  Best practice: Store templates in src/main/resources/ and reference by classpath. Keeps business copy out of flow XML and lets
  non-developers edit it.

  ---
  Custom Business Events
  
  Best for: Business-level audit trail in Anypoint Monitoring — separate from technical logs.

  ┌────────────────────────────┬───────────────────────────────────────────┐
  │          Use Case          │        Key-value pairs to capture         │
  ├────────────────────────────┼───────────────────────────────────────────┤
  │ Track each completed order │ orderId, customerId, totalAmount, channel │
  ├────────────────────────────┼───────────────────────────────────────────┤
  │ Flag a payment retry       │ paymentId, attemptNumber, reason          │
  ├────────────────────────────┼───────────────────────────────────────────┤
  │ Record SLA breach          │ flowName, durationMs, threshold           │
  └────────────────────────────┴───────────────────────────────────────────┘

  Best practice: Use these for business metrics, not technical ones. They appear in the Business Events tab in Runtime Manager and can
  trigger alerts — keep the payload small and meaningful to a business analyst, not just an engineer.

  ---
  Endpoints

  Scheduler

  Best for: Any time-triggered processing that does not depend on an inbound request.

  ┌──────────────────────────────────────────────────────┬────────────────────────────┐
  │                       Use Case                       │       Configuration        │
  ├──────────────────────────────────────────────────────┼────────────────────────────┤
  │ Poll a DB table every 5 minutes for new records      │ Fixed frequency: 5 minutes │
  ├──────────────────────────────────────────────────────┼────────────────────────────┤
  │ Run a nightly reconciliation report at 02:00         │ Cron: 0 0 2 * * ?          │
  ├──────────────────────────────────────────────────────┼────────────────────────────┤
  │ Purge an Object Store of expired sessions every hour │ Cron: 0 0 * * * ?          │
  ├──────────────────────────────────────────────────────┼────────────────────────────┤
  │ Trigger a bulk sync every weekday at 08:00           │ Cron: 0 0 8 ? * MON-FRI    │
  └──────────────────────────────────────────────────────┴────────────────────────────┘

  Best practice: In CloudHub 2.0, only one replica should run a Scheduler at a time. Use the disallowConcurrentExecution setting and
  consider an Object Store lock if you run multiple replicas.

  ---
  Error Handling
  
  On Error Continue vs On Error Propagate

  On Error Continue  →  absorbs the error; flow returns normally to its caller
  On Error Propagate →  re-throws the error; caller sees a failure

  ┌────────────────────────────────────────────────────────────┬────────────────────────────────────────────────────────────────────────┐
  │                          Scenario                          │                                  Use                                   │
  ├────────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
  │ A non-critical enrichment step fails (e.g. loyalty points  │ On Error Continue with a fallback Set Payload                          │
  │ lookup) — default values are acceptable                    │                                                                        │
  ├────────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
  │ A mandatory downstream call fails — the caller must know   │ On Error Propagate                                                     │
  ├────────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
  │ You need to log + send to a dead-letter queue, then        │ On Error Continue (acknowledge) after logging and queuing              │
  │ acknowledge the message                                    │                                                                        │
  ├────────────────────────────────────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
  │ REST API — map any exception to a structured JSON error    │ On Error Continue in the HTTP listener flow's error handler; build     │
  │ response                                                   │ {"error": "...", "code": ...} in a Transform Message                   │
  └────────────────────────────────────────────────────────────┴────────────────────────────────────────────────────────────────────────┘

  Best practice: Define a global error handler for common cases (log + structured response), and use flow-level error handlers only for
  flow-specific compensation logic.

  ---
  Flow Control (Routers)
  
  Choice

  Best for: Conditional branching — exactly one route executes.

  ┌─────────────────────────────────────┬────────────────────────────────────────────────────────┐
  │              Use Case               │                       Condition                        │
  ├─────────────────────────────────────┼────────────────────────────────────────────────────────┤
  │ Route by content type               │ attributes.headers.'Content-Type' == 'application/xml' │
  ├─────────────────────────────────────┼────────────────────────────────────────────────────────┤
  │ VIP vs standard customer processing │ vars.customer.tier == "VIP"                            │
  ├─────────────────────────────────────┼────────────────────────────────────────────────────────┤
  │ Skip processing if payload is empty │ payload == null or payload isEmpty                     │
  └─────────────────────────────────────┴────────────────────────────────────────────────────────┘

  ---
  Scatter-Gather
  
  Best for: Parallel fan-out to independent sources; combine all results. (See previous answer for full detail.)

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                Use Case                                 │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ Aggregate data from Orders + Customers + Inventory APIs in one response │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ Call two pricing engines and return both quotes                         │
  ├─────────────────────────────────────────────────────────────────────────┤
  │ Simultaneously write to a DB and publish an event to a queue            │
  └─────────────────────────────────────────────────────────────────────────┘

  ---
  First Successful
  
  Best for: Fallback chains — try routes in order, return the first one that succeeds.

  ┌──────────────────────────────────────────────┬───────────────────────────────────────────────────┐
  │                   Use Case                   │                    Route order                    │
  ├──────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ Primary cache → fallback to DB               │ Route 1: Object Store lookup; Route 2: DB query   │
  ├──────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ Primary API → fallback to secondary API      │ Route 1: preferred vendor; Route 2: backup vendor │
  ├──────────────────────────────────────────────┼───────────────────────────────────────────────────┤
  │ Try HTTPS → fallback to HTTP (legacy system) │ Route 1: secure endpoint; Route 2: plain endpoint │
  └──────────────────────────────────────────────┴───────────────────────────────────────────────────┘

  ---
  Round Robin

  Best for: Distributing load across equivalent endpoints when no built-in load balancer is available.

  ┌────────────────────────────────────────────────────────────────────┐
  │                              Use Case                              │
  ├────────────────────────────────────────────────────────────────────┤
  │ Spread outbound calls across multiple instances of a legacy system │
  ├────────────────────────────────────────────────────────────────────┤
  │ Distribute work items across a pool of worker queues               │
  └────────────────────────────────────────────────────────────────────┘

  ---
  Scopes

  Try

  Best for: Isolating error handling to a specific section of a flow without creating a separate subflow.

  Flow
    ├── Mandatory steps (no try needed)
    └── Try
          ├── Optional enrichment HTTP call
          └── On Error Continue → Set Variable 'enrichment' to null

  ┌───────────────────────────────────────────────────────────────────┐
  │                             Use Case                              │
  ├───────────────────────────────────────────────────────────────────┤
  │ Wrap a non-critical HTTP call; continue with defaults if it fails │
  ├───────────────────────────────────────────────────────────────────┤
  │ Retry a flaky external call without retrying the entire flow      │
  ├───────────────────────────────────────────────────────────────────┤
  │ Isolate DB writes so failures trigger compensation logic inline   │
  └───────────────────────────────────────────────────────────────────┘

  ---
  For Each
  
  Best for: Iterating over a collection and applying the same processing to every element.

  ┌──────────────────────────────────────────────────────────┬────────────────────────┐
  │                         Use Case                         │       Collection       │
  ├──────────────────────────────────────────────────────────┼────────────────────────┤
  │ Insert each row from a DB result set into another system │ payload (list of maps) │
  ├──────────────────────────────────────────────────────────┼────────────────────────┤
  │ Send one notification per item in an order               │ payload.lineItems      │
  ├──────────────────────────────────────────────────────────┼────────────────────────┤
  │ Validate and transform each element of a JSON array      │ Any array payload      │
  └──────────────────────────────────────────────────────────┴────────────────────────┘

  Best practice: For Each processes sequentially. If order does not matter and throughput is important, use Parallel For Each (also a core
  scope in Mule 4.2+) or a Batch Job for very large sets.

  ---
  Async
  
  Best for: Fire-and-forget side effects that should not block the main response.

  ┌──────────────────────────────────────────────────────────────────────┐
  │                               Use Case                               │
  ├──────────────────────────────────────────────────────────────────────┤
  │ Write an audit log to a DB after responding to the client            │
  ├──────────────────────────────────────────────────────────────────────┤
  │ Publish a domain event to a queue without delaying the HTTP response │
  ├──────────────────────────────────────────────────────────────────────┤
  │ Trigger a slow notification (SMS, email) in the background           │
  └──────────────────────────────────────────────────────────────────────┘

  Best practice: The Async scope runs in a separate thread. Variables and payload changes inside it do not affect the parent flow. Do not
  use it when you need the result.

  ---
  Cache
  
  Best for: Storing the result of an expensive operation and reusing it for identical inputs within a TTL window.

  ┌────────────────────────────────────────────────────────────────┬───────────────────────────────────────┐
  │                            Use Case                            │                  TTL                  │
  ├────────────────────────────────────────────────────────────────┼───────────────────────────────────────┤
  │ Cache a reference data lookup (country codes, product catalog) │ 1 hour                                │
  ├────────────────────────────────────────────────────────────────┼───────────────────────────────────────┤
  │ Cache an OAuth token until near-expiry                         │ Token lifetime − 60 s                 │
  ├────────────────────────────────────────────────────────────────┼───────────────────────────────────────┤
  │ Cache the result of an identical Scatter-Gather aggregation    │ 30 seconds for high-traffic endpoints │
  └────────────────────────────────────────────────────────────────┴───────────────────────────────────────┘

  Best practice: Define a cache key strategy using a DataWeave expression that captures all inputs that determine a unique result (e.g.
  #[vars.customerId ++ vars.region]).

  ---
  Until Successful
  
  Best for: Retrying a single operation until it succeeds or a maximum attempt count is reached.

  ┌─────────────────────────────────────────────────────────┬─────────────┬──────────────────────────────┐
  │                        Use Case                         │ Max retries │ Milliseconds between retries │
  ├─────────────────────────────────────────────────────────┼─────────────┼──────────────────────────────┤
  │ Call a flaky external API that occasionally returns 503 │ 3           │ 2000                         │
  ├─────────────────────────────────────────────────────────┼─────────────┼──────────────────────────────┤
  │ Write to a DB that may be momentarily locked            │ 5           │ 1000                         │
  ├─────────────────────────────────────────────────────────┼─────────────┼──────────────────────────────┤
  │ Send to a queue that may be temporarily unavailable     │ 4           │ 3000                         │
  └─────────────────────────────────────────────────────────┴─────────────┴──────────────────────────────┘

  Best practice: Wrap only the single operation that may fail (e.g. just the http:request), not the surrounding transformation logic.
  Combine with a Try inside to control which error types trigger a retry vs. an immediate failure.

  ---
  Transformers
  
  ┌────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
  │   Component    │                                                      Use Case                                                      │
  ├────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Set Payload    │ Assign a static or simple expression value — stub responses, test harnesses, default fallback values               │
  ├────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Set Variable   │ Store an intermediate value you will need later (e.g. save original payload before Transform, store a correlation  │
  │                │ key)                                                                                                               │
  ├────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
  │ Remove         │ Clean up sensitive variables (credentials, tokens) before a flow logs its final state or passes control to an      │
  │ Variable       │ external subflow                                                                                                   │
  └────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  Best practice for variables: Set variables to preserve original payload before any transformation — vars.originalPayload = payload — so
  you can reference it downstream without re-fetching.
