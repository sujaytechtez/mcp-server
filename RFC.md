# RFC-0001: MCP Server – Python SDK v1.0



Status: Accepted

Version: 1.0

Author: Project MCP

Last Updated: 2026-01-05



## 1\. Abstract



This RFC defines MCP Server v1.0, a Python SDK for exposing structured, discoverable, and policy-controlled capabilities (“tools”) to AI agents via the Model Context Protocol (MCP).



The MCP Server SDK provides a stable, enterprise-oriented contract between agents and backend systems, emphasizing safety, observability, and vendor neutrality.



## 2\. Motivation



AI agents increasingly interact with real systems (databases, services, files, workflows).

Today, this interaction is typically implemented using ad-hoc tool wrappers that suffer from:



Lack of standard discovery



Weak input/output contracts



No agent identity



No policy enforcement



No observability or auditability



This makes production deployment risky, especially in regulated or enterprise environments.



MCP Server addresses this gap by providing a protocol-first, agent-agnostic server SDK that cleanly separates:



Agent reasoning



Capability exposure



Execution control



## 3\. Goals



The MCP Server SDK v1.0 aims to:



Provide a stable API for defining agent-callable tools



Enforce strict input/output schemas



Make agent identity explicit



Enable policy-based access control



Offer observable execution hooks



Remain model- and framework-neutral



## 4\. Non-Goals



MCP Server explicitly does NOT:



Orchestrate agents



Manage prompts or reasoning



Chain tools together



Provide LLM integrations



Act as an agent framework



Enforce authentication standards (v1)



These concerns are intentionally out of scope.



## 5\. Terminology

Term	Definition

Tool	A callable capability exposed to agents

Agent	Any caller that invokes tools via MCP

Capability	Metadata describing available tools

Policy	A decision rule allowing or denying execution

Execution	A single tool invocation lifecycle

## 6\. Architecture Overview

Agent (any model / framework)

```text
        ↓
   MCP Client
        ↓
┌────────────────────────┐
│   MCP Server (SDK)     │
│ ─ Tool Registry        │
│ ─ Schema Validation    │
│ ─ Policy Engine        │
│ ─ Execution Hooks      │
│ ─ Transport Adapter    │
└────────────────────────┘
        ↓
   Backend Systems
```


The MCP Server owns tool exposure, validation, and control, not agent behavior.



## 7\. Core Abstraction: McpServer

Construction

```python
McpServer(
  \*,
name: str,
version: str,
transport: str = "http",
description: str | None = None,
)
```



### Responsibilities



Register tools



Expose capabilities



Enforce policies



Execute tools



Emit lifecycle events



Bind transport



The server must not depend on global state.



## 8\. Tool Definition

Tool Decorator

```python
@tool(
    *,
    name: str,
    description: str | None = None,
    timeout_ms: int = 1000,
    idempotent: bool = True,
)
```




### Requirements



Input MUST be a Pydantic BaseModel



Output MUST be a Pydantic BaseModel



Tools MUST be synchronous in v1.0



### Example

```python
class CustomerRequest(BaseModel):
    customer_id: str


class CustomerResponse(BaseModel):
    customer_id: str
    status: str


@tool(name="get_customer")
def get_customer(req: CustomerRequest) -> CustomerResponse:
    ...
```


## 9\. Tool Metadata



Each registered tool produces immutable metadata:



```python
class ToolMetadata(BaseModel):
    name: str
    description: str | None
    input_schema: dict
    output_schema: dict
    timeout_ms: int
    idempotent: bool
```





This metadata is returned via capability discovery.



## 10\. Agent Context

Definition

```python
class AgentContext:
    agent_id: str
    model: str | None
    request_id: str
    metadata: dict[str, str]
```




### Injection Rule



If a tool declares an AgentContext parameter, it is automatically injected.



Tools MUST NOT read transport headers or raw requests directly.



## 11\. Policy Engine

### Policy Decision

PolicyDecision.allow()

PolicyDecision.deny(reason: str)



### Policy Hook Signature

(AgentContext, tool\_name: str, args: dict) -> PolicyDecision



### Semantics



Policies are evaluated before execution



All policies must allow execution



First denial short-circuits execution



## 12\. Execution Lifecycle Hooks

### Hook Points



on\_execute\_start



on\_execute\_end



on\_execute\_error



### Guarantees



Start hook always fires



Exactly one terminal hook fires



Hooks cannot mutate results



These hooks enable logging, tracing, auditing, and metrics.



## 13\. Wire Protocol

### Capability Discovery

GET /mcp/capabilities





### Response:



```json
{
  "server": "customer-mcp",
  "version": "1.0.0",
  "tools": [...]
}
```



### Tool Execution

POST /mcp/execute





### Request:



```json
{
  "tool": "get_customer",
  "arguments": {
    ...
  }
}
```



## 14\. Error Model

### Error Shape

```json
{
  "error": "POLICY_DENIED",
  "message": "Blocked in prod"
}
```



### Reserved Error Codes



INVALID\_INPUT



TOOL\_NOT\_FOUND



POLICY\_DENIED



EXECUTION\_ERROR



TIMEOUT



## 15\. Transport



v1.0 supports:



HTTP (required)



WebSocket (optional)



STDIO (optional)



Transport choice must not affect semantics.



## 16\. Versioning \& Compatibility



This RFC defines v1.0



Breaking changes require a major version bump



Wire contract stability is a hard requirement



## 17\. Security Considerations



Tools execute arbitrary code and must be explicitly registered



Policies are mandatory for production use



Agent identity must be treated as untrusted input



Observability hooks should be enabled in regulated environments



## 18\. Future Work (Non-Normative)



Async tool execution



Authentication standards



Rate limiting



Secrets management



Policy DSLs



Tool composition



## 19\. Conclusion



MCP Server v1.0 establishes a minimal, stable, and enterprise-ready foundation for exposing capabilities to AI agents.



It intentionally prioritizes control, clarity, and correctness over orchestration or convenience.

