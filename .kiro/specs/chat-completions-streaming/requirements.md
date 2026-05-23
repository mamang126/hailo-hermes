# Requirements Document

## Project Description (Input)
i need to fix an error that have this proxy of hailo-ollama.
The error that I have when I use it with Hermes is that calling /vi/chat/completions doesn't support streaming, and we have to use /v1/chat instead.

I need that /v1/chat/completions endpoint have streaming compatibility. We have two ways of solving this:
- Redirect to chat endpoint that have streaming
- Fake streaming in the proxy itself

Get the better solution for this fix.

## Introduction

This document defines the requirements for adding Server-Sent Events (SSE) streaming compatibility to the `/v1/chat/completions` endpoint in the Hailo-Hermes proxy. The proxy currently handles non-streaming responses only; clients (e.g., Home Assistant's `extended_openai_conversation`) may send requests with `"stream": true`, which currently fails or returns an incompatible response. The chosen approach is **fake streaming in the proxy**: the proxy strips the `stream` flag before forwarding to hailo-ollama, waits for the full response, then emits it as a single-chunk SSE stream. This preserves the full response-rewriting pipeline (tool call rewriting, follow-up truncation, retry logic) which requires the complete response body before any transformation.

## Boundary Context

- **In scope**: Handling `"stream": true` on `POST /v1/chat/completions`; emitting well-formed OpenAI SSE format; preserving all existing request/response transformations.
- **Out of scope**: True token-by-token streaming to hailo-ollama; streaming on native `/api/chat` or `/api/generate` endpoints; changes to non-streaming behaviour.
- **Adjacent expectations**: All existing proxy behaviours (tool call injection, sanitisation, retry, logging) must continue to operate identically for both streaming and non-streaming requests.

## Requirements

### Requirement 1: Streaming Request Detection

**Objective:** As a proxy, I want to detect when a client requests a streaming response, so that the appropriate response path is selected.

#### Acceptance Criteria
1. When a `POST /v1/chat/completions` request body contains `"stream": true`, the Proxy shall treat the request as a streaming request.
2. When a `POST /v1/chat/completions` request body contains `"stream": false` or omits the `stream` field, the Proxy shall treat the request as a non-streaming request and follow the existing response path without change.
3. The Proxy shall detect the `stream` field after `fix_json_control_chars` has been applied but before forwarding to the backend.

### Requirement 2: Stream Flag Removal Before Forwarding

**Objective:** As a proxy, I want to strip the `stream` flag from the request before sending it to hailo-ollama, so that the backend receives a valid non-streaming request it can fulfill.

#### Acceptance Criteria
1. When a streaming request is detected, the Proxy shall remove the `"stream"` key from the JSON body before forwarding to hailo-ollama.
2. The Proxy shall apply all existing request transformations (`inject_tool_prompt`, `sanitize_conversation_roles`, `sanitize_for_hailo`, `inject_defaults`) to the modified body in the same order as for non-streaming requests.
3. The Proxy shall forward the modified (non-streaming) request to the backend and collect the complete response body before responding to the client.

### Requirement 3: SSE Response Emission

**Objective:** As a proxy, I want to emit the backend response as a well-formed OpenAI SSE stream, so that streaming-capable clients receive a protocol-compliant response.

#### Acceptance Criteria
1. When the backend returns a successful response to a streaming request, the Proxy shall respond with HTTP status 200 and `Content-Type: text/event-stream`.
2. The Proxy shall emit the response as exactly two SSE events: one `data:` line containing the transformed response chunk, followed by one `data: [DONE]` terminator line.
3. The SSE chunk emitted in the `data:` line shall follow the OpenAI streaming chunk format: `{"id": "...", "object": "chat.completion.chunk", "choices": [{"index": 0, "delta": {"role": "assistant", "content": "..."}, "finish_reason": "stop"}]}`.
4. When the transformed response contains `finish_reason: "tool_calls"` (i.e. a tool call was rewritten by `rewrite_tool_response`), the Proxy shall set `finish_reason: "tool_calls"` and include `"delta": {"role": "assistant", "tool_calls": [...]}` in the SSE chunk.
5. The Proxy shall not set a `Content-Length` header on streaming responses, as the body length is not known in advance.

### Requirement 4: Response Pipeline Preservation

**Objective:** As a proxy, I want all existing response-rewriting logic to run on streaming requests, so that tool calls, retries, and follow-up truncation behave identically regardless of the stream flag.

#### Acceptance Criteria
1. The Proxy shall apply `rewrite_tool_response` to the full backend response before constructing the SSE chunk, when `tool_mode` is `True`.
2. The Proxy shall apply retry-on-rejection logic (when `ARGS.retry_on_rejection` is enabled and rewrite status is `'rejected'`) before constructing the SSE chunk.
3. The Proxy shall apply `truncate_followup_response` to the full backend response when `tool_mode` is `'followup'` before constructing the SSE chunk.
4. The Proxy shall apply debug logging (`_debug_log`) for both the forwarded request and the backend response on streaming requests, consistent with non-streaming behaviour.

### Requirement 5: Error Handling for Streaming Responses

**Objective:** As a proxy, I want streaming error conditions to be handled gracefully, so that clients receive a meaningful error instead of a broken stream.

#### Acceptance Criteria
1. If the backend returns an HTTP error on a streaming request, the Proxy shall respond with the same HTTP status code and error body as it would for a non-streaming request.
2. If a transport-level exception occurs during a streaming request, the Proxy shall respond with HTTP 502 and the plain-text body `Bad Gateway`, consistent with existing error handling.
3. If the full backend response body cannot be parsed as JSON during SSE construction, the Proxy shall emit the raw response content in the `delta.content` field of the SSE chunk rather than failing silently.
