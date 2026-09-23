# Pick-6 Interception — Design Specification

**Date:** 2026-09-23  
**Status:** Engineering-reviewed design draft  
**Repository:** `AlkaiDynamics/Pick-6-Interception`

> This document records the intended architecture. It is not represented as technically certified by the project owner. Engineering correctness must be established by implementation tests, failure-injection tests, and runtime evidence.

## 1. Goal

Provide a reusable, zero-paid-inference bridge for agentic projects that can:

1. **Assess** how a project performs model inference and how its workflow can safely pause.
2. **Intercept** supported inference calls before they reach a paid provider.
3. **Package** each inference request into a self-contained prompt optimized for manual fulfillment by ChatGPT.
4. **Hold** the calling workflow without treating delayed inference as an error.
5. **Return** a validated response through a separate return channel.
6. **Resume** the suspended workflow exactly once, preserving state and avoiding duplicate execution.

These six stages are the V1 functional core: **Assess → Intercept → Package → Hold → Return → Resume**.

The system must prioritize recoverability over speed. A machine restart, bridge restart, duplicate response, partial file write, or long human delay must not silently lose or duplicate work.

## 2. Constraints

- Paid model APIs are optional and disabled by default.
- Existing projects may use different agent frameworks, SDKs, synchronous or asynchronous execution, retries, streaming, or custom wrappers.
- The bridge must not assume that a long-lived HTTP request can remain open indefinitely.
- The bridge must not blindly rewrite an unfamiliar project.
- Human-visible exchange must use two operational directories: `CATCH/` and `RETURN/`.
- Internal transactional state may use SQLite beside those directories.
- Credentials, authorization headers, `.env` secrets, tokens, and unrelated sensitive data must never be copied into prompt packets.
- Project-owner approval is not a substitute for technical verification.

## 3. Critical Design Correction

An OpenAI-compatible proxy is useful, but **cannot provide durable suspension for every arbitrary caller by itself**.

A normal synchronous call such as:

```python
response = client.chat.completions.create(...)
```

may be governed by SDK, HTTP, load-balancer, framework, job-runner, or operating-system timeouts. Keeping the local connection open is therefore only safe when assessment confirms that the call path can tolerate the expected delay.

Pick-6 consequently supports two hold modes:

### 3.1 Blocking Hold

Used when the caller can safely remain alive.

```text
caller -> local proxy -> CATCH -> wait -> RETURN -> local proxy -> caller
```

The bridge keeps the local request pending and wakes it when the matching response is committed.

This mode is the lowest-friction compatibility path but is not considered crash-durable.

### 3.2 Durable Suspend

Used when the workflow supports checkpoint/restart or can be adapted at the inference boundary.

```text
caller
  -> inference adapter
  -> persist continuation/checkpoint
  -> CATCH
  -> controlled SUSPENDED state

RETURN
  -> validate
  -> mark response ready
  -> restore/restart workflow at blocked step
  -> inject response
  -> continue
```

This is the preferred mode for long human delays and workflows that must survive process or machine restarts.

## 4. High-Level Architecture

```text
                    +---------------------------+
                    |     Existing Project      |
                    | agents / workflows / jobs |
                    +-------------+-------------+
                                  |
                                  | inference request
                                  v
                    +---------------------------+
                    |   Interception Boundary   |
                    | proxy or SDK adapter      |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    |     Request Normalizer    |
                    | classify + redact + pack  |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    |    Durable State Ledger   |
                    |          SQLite           |
                    +-------------+-------------+
                                  |
                           atomic publish
                                  |
                                  v
                    .inference_bridge/CATCH/
                                  |
                         ChatGPT handoff
                                  |
                                  v
                    .inference_bridge/RETURN/
                                  |
                           atomic consume
                                  |
                                  v
                    +---------------------------+
                    | Response Validator/Router |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    |      Resume Controller    |
                    +-------------+-------------+
                                  |
                                  v
                    +---------------------------+
                    |     Existing Project      |
                    +---------------------------+
```

## 5. Repository / Runtime Layout

For an integrated project:

```text
<PROJECT>/
└── .inference_bridge/
    ├── CATCH/
    │   ├── req_<uuid>.json
    │   └── INFERENCE_BATCH.md
    ├── RETURN/
    │   └── req_<uuid>.json
    ├── archive/
    │   ├── completed/
    │   ├── failed/
    │   └── cancelled/
    ├── assessment.json
    ├── bridge.toml
    └── state.sqlite
```

`CATCH/` and `RETURN/` are the only directories required for human exchange. The other files exist for safety, configuration, auditability, and recovery.

All queue writes use **write-to-temp + fsync where practical + atomic rename**. A watcher must never treat a partially written file as complete.

## 6. Project Assessor

Command:

```text
pick6 assess <project-path>
```

The assessor is read-only. It inventories the project before any interception changes are proposed.

### 6.1 Detection Targets

The assessor searches for:

- OpenAI-compatible SDK usage
- Anthropic SDK usage
- LiteLLM
- LangChain / LangGraph
- CrewAI
- AutoGen
- LlamaIndex
- direct `requests`, `httpx`, or `aiohttp` model calls
- custom model-provider wrappers
- model-related environment variables and configuration keys
- retry policies
- request timeouts
- streaming calls
- async task queues and job runners
- workflow checkpoint/state facilities
- call sites that already centralize inference

Detection is heuristic. A detected framework is evidence for a possible adapter, not proof that all calls flow through it.

### 6.2 Assessment Output

Example:

```json
{
  "schema_version": "1.0",
  "project": "example",
  "frameworks": ["crewai", "litellm"],
  "providers": ["openai"],
  "call_sites": [
    {
      "file": "agents/researcher.py",
      "line": 81,
      "mechanism": "litellm.completion",
      "confidence": "high"
    }
  ],
  "streaming_detected": false,
  "checkpoint_support": true,
  "recommended_interception": "openai_compatible_proxy",
  "recommended_hold_mode": "durable_suspend",
  "unresolved": []
}
```

### 6.3 Mutation Gate

`assess` never patches source code.

A separate command, such as:

```text
pick6 install <project-path>
```

may generate configuration or adapters only after the assessment identifies a supported integration path. Unsupported or ambiguous paths fail closed with a diagnostic report rather than speculative edits.

## 7. Interception Boundary

V1 supports two primary interception mechanisms.

### 7.1 OpenAI-Compatible Local Proxy

Example endpoint:

```text
http://127.0.0.1:8742/v1/chat/completions
```

The existing application points its configurable model `base_url` at Pick-6. The proxy accepts the supported subset of the OpenAI-compatible request shape, normalizes it, emits a CATCH packet, waits when blocking mode is permitted, and returns a compatible response.

The proxy must preserve caller-visible fields that downstream code may depend on, including request identity, response role/content, tool-call structure when supported, and usage fields where a harmless zero/unknown representation is accepted.

Unsupported protocol features must be rejected explicitly; they must not be silently approximated.

### 7.2 SDK / Framework Adapter

For durable suspension, Pick-6 integrates before the HTTP boundary or at a framework-supported model abstraction.

The adapter is responsible for:

1. creating an immutable inference request ID;
2. persisting enough workflow state to resume safely;
3. emitting the request packet;
4. transitioning the workflow into a controlled suspended state;
5. later receiving the validated result;
6. resuming at the blocked inference step without re-running prior side effects.

Adapters remain small and framework-specific. Core queue, ledger, validation, redaction, and prompt-packing logic stays framework-independent.

### 7.3 Excluded From V1: Transparent Network MITM

OS-level packet interception, TLS man-in-the-middle interception, DNS redirection, and arbitrary process traffic capture are excluded from V1. They add certificate, security, protocol, debugging, and accidental-capture risks without solving the durable-resume problem.

## 8. Request Lifecycle and State Machine

Canonical request states:

```text
NEW
  -> PACKAGING
  -> QUEUED
  -> WAITING_FOR_INFERENCE
  -> RESPONSE_DETECTED
  -> VALIDATING
  -> READY_TO_RESUME
  -> RESUMING
  -> COMPLETED
```

Terminal exceptional states:

```text
FAILED
CANCELLED
QUARANTINED
```

Allowed recovery transitions are explicit. For example, a malformed RETURN packet moves the request to `QUARANTINED`; it does not wake the caller.

The SQLite ledger is authoritative for machine state. Queue files are exchange artifacts, not the sole source of truth.

## 9. Idempotency and Exactly-Once Resume

Human-delayed workflows are especially exposed to duplicate actions. Pick-6 therefore uses:

- immutable `request_id` values;
- a unique ledger row per request;
- response-content hash recording;
- atomic state transitions;
- a resume lease / compare-and-swap transition;
- explicit completion receipts;
- rejection or quarantine of conflicting duplicate responses.

A repeated identical RETURN file is harmless. A second different response for an already completed request is retained for diagnosis but must not trigger another resume.

Pick-6 can guarantee **exactly-once bridge resume transition** under its own ledger. It cannot guarantee exactly-once external side effects unless the surrounding workflow is itself idempotent or checkpointed before those effects.

## 10. CATCH Packet

Each request becomes a self-contained machine-readable packet.

```json
{
  "schema_version": "1.0",
  "request_id": "019abc...",
  "created_at": "2026-09-23T00:00:00Z",
  "state": "WAITING_FOR_INFERENCE",

  "project": {
    "name": "Archotraz",
    "root_label": "project-root"
  },

  "caller": {
    "agent": "architecture_reviewer",
    "workflow": "candidate_universe",
    "step": "review_patch"
  },

  "inference": {
    "capability": "reasoning",
    "original_provider": "openai",
    "original_model": "example-model",
    "temperature": 0.2,
    "streaming_requested": false
  },

  "instructions": {
    "system": "...",
    "developer": "...",
    "messages": []
  },

  "context": {
    "working_directory_label": "...",
    "relevant_files": [],
    "artifacts": [],
    "previous_outputs": []
  },

  "task": {
    "objective": "...",
    "required_output": "...",
    "constraints": [],
    "acceptance_criteria": []
  },

  "response_contract": {
    "format": "json",
    "schema": {},
    "must_preserve": []
  },

  "chatgpt_prompt": "..."
}
```

### 10.1 Prompt Packing

`chatgpt_prompt` is derived from the original call plus project context. Its objective is to make manual fulfillment possible without reconstructing the workflow mentally.

The packer should preserve, in order:

1. authoritative system/developer instructions;
2. task objective;
3. required output contract;
4. relevant conversation/history needed for the decision;
5. minimum sufficient project artifacts;
6. explicit constraints;
7. acceptance criteria;
8. request identity and return instructions.

The packer may summarize redundant context only when the original material remains referenced and the summarization cannot alter a required exact value.

### 10.2 Secret Redaction

Before publication to CATCH, the bridge removes or masks at minimum:

- `Authorization` headers;
- API keys/tokens;
- passwords;
- private keys;
- cookie/session secrets;
- `.env` values unless explicitly allowlisted as non-secret;
- provider credentials embedded in URLs.

Redaction happens before any optional remote mailbox transport.

## 11. RETURN Packet

Minimum successful response:

```json
{
  "schema_version": "1.0",
  "request_id": "019abc...",
  "status": "COMPLETED",
  "response": {
    "role": "assistant",
    "content": "..."
  }
}
```

For tool-calling or structured responses, the packet may also carry typed fields required by the original call contract.

A RETURN packet is not eligible to wake a workflow until:

- `request_id` exists and is waiting;
- schema version is supported;
- response shape satisfies the recorded response contract;
- the request is not already terminal;
- no conflicting accepted response exists.

## 12. ChatGPT Handoff Boundary

Pick-6 can create and watch a local filesystem mailbox. ChatGPT in a normal browser conversation cannot autonomously monitor an arbitrary local Windows directory.

Therefore the transport between `CATCH/RETURN` and ChatGPT is a separate boundary.

### 12.1 V1 Local/Manual Transport

The bridge creates both individual JSON packets and a compact batch manifest:

```text
CATCH/INFERENCE_BATCH.md
```

The user supplies the pending batch to ChatGPT through an available file/repository/workspace mechanism. ChatGPT produces matching RETURN packets, which are placed back into `RETURN/`.

This requires no paid model API and preserves the core hold/resume architecture.

### 12.2 Pluggable Mailbox Transport

The core defines a transport interface so a later adapter can synchronize CATCH and RETURN with a location ChatGPT can access, without changing agent integrations.

Potential transports include a private cloud-synced directory or another authenticated document/file connector. Transport selection must preserve privacy and must never publish project prompts into a public repository by default.

The `Pick-6-Interception` source repository being public is **not** an acceptable runtime mailbox for potentially sensitive inference packets.

## 13. Batch Manifest

To conserve human interaction overhead, the bridge maintains a generated summary of pending requests:

```text
PENDING: 4

[req-001] Archotraz — architecture review
[req-002] Rosettas Tones — source classification
[req-003] File Atlas — dedupe decision
[req-004] SHARER — planning step
```

Independent requests may be fulfilled together. Each result remains a separate RETURN packet and is validated independently.

The manifest is an index, not authoritative state.

## 14. SQLite Ledger

Suggested tables:

### `requests`

- `request_id` primary key
- `project_id`
- `caller_id`
- `state`
- `hold_mode`
- `created_at`
- `updated_at`
- `catch_path`
- `response_contract_json`
- `request_hash`
- `checkpoint_ref`
- `error_code`
- `error_detail`

### `responses`

- `response_id` primary key
- `request_id`
- `received_at`
- `response_hash`
- `return_path`
- `validation_status`
- `validation_error`
- `accepted`

### `events`

Append-only lifecycle events for diagnosis and crash recovery.

SQLite should use transactions for all state transitions. WAL mode may be enabled where filesystem semantics are suitable.

## 15. CLI Surface

V1 target commands:

```text
pick6 assess <project>
pick6 install <project>
pick6 serve <project>
pick6 status <project>
pick6 pending <project>
pick6 validate-return <file>
pick6 resume <request-id>
pick6 recover <project>
```

`resume` must be idempotent and normally invoked by the watcher rather than manually.

## 16. Error Handling

### Missing RETURN

No timeout is treated as inference failure by default. The request remains waiting until explicitly cancelled or a configured administrative expiry is reached.

### Malformed RETURN

Quarantine the response, record the validation error, leave the workflow suspended, and regenerate the batch manifest with the error noted.

### Unknown Request ID

Quarantine. Never create state implicitly from a RETURN packet.

### Duplicate RETURN

If identical to the accepted response, ignore safely. If conflicting, quarantine and require explicit resolution.

### Bridge Restart

On startup, reconcile SQLite state with CATCH and RETURN files. Resume any request that is `READY_TO_RESUME`; recreate missing exchange artifacts when they can be deterministically reconstructed from ledger data.

### Machine Restart

Blocking holds are lost and must be reported as interrupted. Durable-suspend requests remain recoverable from checkpoint + ledger.

### Project Changed While Waiting

The request records a project/workflow revision fingerprint when available. Before durable resume, the adapter compares the relevant revision. A material mismatch pauses in `QUARANTINED` rather than injecting an answer into a semantically different workflow.

## 17. Concurrency

V1 must support multiple projects and multiple simultaneous pending requests.

Rules:

- request IDs are globally unique;
- each request has one accepted response;
- queue scanning never assumes filename ordering equals execution ordering;
- one request may resume while others remain waiting;
- per-project concurrency limits are configurable;
- resume operations use a lease so two watchers cannot wake the same request concurrently.

## 18. Security Boundary

The local proxy binds to loopback (`127.0.0.1`) by default.

Remote exposure is disabled unless explicitly configured.

The bridge must not log raw secrets. Diagnostic logging defaults to metadata and redacted prompt fragments.

Runtime CATCH and RETURN data should be excluded from source control by generated `.gitignore` rules unless a user intentionally overrides this for a private repository.

## 19. Testing Strategy

Engineering correctness is established by executable evidence, not by owner review of implementation details.

### 19.1 Unit Tests

- framework/call-site detection
- secret redaction
- prompt-packet construction
- schema validation
- atomic queue publication
- request state transitions
- duplicate-response handling
- resume lease behavior
- manifest generation

### 19.2 Integration Tests

- fake OpenAI-compatible client -> proxy -> CATCH -> RETURN -> compatible client response
- SDK adapter -> durable suspend -> process restart -> RETURN -> resume
- multiple simultaneous requests
- malformed return packet
- duplicate identical return packet
- conflicting duplicate return packet
- bridge restart while waiting
- return arriving before watcher startup

### 19.3 Failure Injection

Kill the bridge at each state transition and verify recovery.

At minimum:

- during CATCH write;
- after CATCH publication but before ledger update/reconciliation;
- while waiting;
- during RETURN ingestion;
- immediately before resume lease acquisition;
- immediately after resume begins;
- after workflow completion but before archival.

### 19.4 Acceptance Evidence

V1 is not complete until tests demonstrate:

1. no paid-provider request is emitted in interception mode;
2. a supported caller receives a manually supplied response in its expected shape;
3. a durable-suspend workflow survives bridge restart;
4. a durable-suspend workflow survives machine/process restart when its underlying framework supports checkpoint restoration;
5. duplicate RETURN files do not duplicate workflow execution;
6. malformed responses do not wake workflows;
7. secrets planted in a synthetic request do not appear in CATCH output;
8. at least two concurrent pending requests can be independently returned and resumed;
9. unsupported project patterns fail closed with useful diagnostics.

## 20. Non-Goals for V1

- transparent TLS interception;
- arbitrary packet capture;
- a graphical UI;
- autonomous paid fallback;
- replacing an agent framework's entire scheduler;
- pretending arbitrary synchronous Python stacks can be crash-resumed without integration support;
- publishing inference packets to the public Pick-6 source repository;
- solving every provider-specific streaming/tool protocol in the first release.

## 21. Future Extensions

After V1 is proven:

```text
request
  -> deterministic/cached answer if available
  -> local model if sufficient
  -> ChatGPT mailbox
  -> optional paid provider only when explicitly enabled
```

Possible additions:

- local-model routing;
- semantic caching;
- request deduplication across agents;
- priority queues;
- token/context compaction;
- private connector-backed mailbox transports;
- framework-specific adapters beyond the first supported set;
- a lightweight queue dashboard.

These extensions must not alter the core request identity, ledger, or hold/resume semantics.

## 22. Design Decision Summary

**Selected:**

- framework-agnostic core;
- read-only project assessment before mutation;
- OpenAI-compatible proxy for low-friction interception;
- SDK/framework adapter for durable suspension;
- `CATCH/` and `RETURN/` exchange directories;
- SQLite transactional ledger;
- atomic file publication;
- explicit state machine;
- idempotent response ingestion and resume;
- structured ChatGPT-oriented prompt packets;
- secret redaction;
- batch manifest;
- pluggable ChatGPT mailbox transport boundary.

**Rejected for V1:**

- transparent network MITM;
- assuming an HTTP connection can remain alive indefinitely;
- treating filesystem files alone as authoritative transactional state;
- silently modifying unsupported projects;
- public-repository runtime mailboxes.

## 23. Next Engineering Gate

Before implementation, convert this design into a bounded implementation plan with explicit files, tests, dependency choices, and incremental checkpoints.

The project owner should only need to confirm that the **behavioral intent** remains correct: agent inference is captured, packaged for ChatGPT, safely held, returned, and resumed without paid inference. Technical correctness is to be demonstrated by the test suite and runtime evidence.