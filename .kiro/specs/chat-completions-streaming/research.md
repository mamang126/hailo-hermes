# Research & Design Decisions

---
**Purpose**: Capture discovery findings, architectural investigations, and rationale that inform the technical design.

---

## Summary
- **Feature**: `chat-completions-streaming`
- **Discovery Scope**: Extension (existing proxy system, single-file modification)
- **Key Findings**:
  - hailo-ollama does not support SSE streaming on `/v1/chat/completions`; fake streaming (buffer full response, re-emit as SSE) is the correct approach.
  - All existing response-rewriting transformations (tool call rewriting, retry-on-rejection, follow-up truncation) require the **complete** response body — true token-by-token streaming would bypass this pipeline or require buffering it anyway, yielding no net benefit.
  - The OpenAI streaming chunk format (`chat.completion.chunk`) is well-defined and a single chunk with `finish_reason` set is accepted by conforming clients, including Home Assistant's `extended_openai_conversation`.

## Research Log

### OpenAI SSE Streaming Protocol
- **Context**: Need to emit a protocol-compliant streaming response to clients that request `"stream": true`.
- **Sources Consulted**: OpenAI API reference for chat completions streaming (https://platform.openai.com/docs/api-reference/chat/create); Home Assistant `extended_openai_conversation` integration source.
- **Findings**:
  - Each SSE event is `data: <JSON>\n\n`.
  - The stream terminates with `data: [DONE]\n\n`.
  - For content responses, the `delta` object carries `{"role": "assistant", "content": "..."}` in the first chunk and `{}` in the final stop chunk.
  - For tool-call responses, the `delta` carries `{"role": "assistant", "tool_calls": [...]}`.
  - `finish_reason` is `null` on intermediate chunks and `"stop"` / `"tool_calls"` on the final chunk.
  - A single-chunk stream (content + finish_reason in one event) is protocol-valid and accepted by HA.
- **Implications**: A two-event SSE emission (content event + stop event + [DONE]) maximises client compatibility; the extra empty-delta stop event is cheap and signals unambiguous completion.

### Redirect-to-/v1/chat Alternative
- **Context**: The user noted that hailo-ollama's `/v1/chat` endpoint supports streaming.
- **Findings**:
  - Redirecting would require understanding hailo-ollama's `/v1/chat` response format and translating it to the OpenAI SSE format.
  - The existing pipeline runs on full OpenAI-format response bodies — adapting it to consume a streaming `/v1/chat` response would require partial-parse logic and significantly more code.
  - Retry-on-rejection cannot function without the full first-attempt response being available before streaming begins.
  - No documented contract for hailo-ollama's `/v1/chat` endpoint is available.
- **Implications**: Redirect approach would introduce significant complexity and risk for no observable benefit. Rejected.

### Python `http.server` and Chunked / SSE Responses
- **Context**: The proxy uses `http.server.BaseHTTPRequestHandler`; need to verify SSE emission without `Content-Length`.
- **Findings**:
  - `BaseHTTPRequestHandler._send()` currently always sets `Content-Length`. Omitting `Content-Length` and setting `Transfer-Encoding: chunked` (or no encoding) is valid for HTTP/1.1.
  - For SSE, the simplest correct approach is: omit `Content-Length`, write all SSE bytes at once in `wfile.write()`. The connection closes after the write, which is the correct EOF signal for non-persistent streaming clients.
  - No additional dependency required; stdlib `http.server` supports this natively.
- **Implications**: `_send_streaming()` is a small variant of `_send()` without the `Content-Length` header.

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations |
|--------|-------------|-----------|---------------------|
| Fake streaming (chosen) | Strip `stream` flag, collect full response, emit as SSE | Preserves full pipeline; zero new dependencies; minimal code change | Client sees full latency before first token (no real streaming UX) |
| Redirect to `/v1/chat` | Forward to hailo's streaming endpoint, translate to OpenAI SSE | Real streaming UX | Unknown response format; breaks rewrite pipeline; high complexity |
| Proxy-side true streaming | Pipe hailo chunks to client in real-time | Real streaming UX | Requires hailo format knowledge; incompatible with tool-call rewriting |

## Design Decisions

### Decision: Fake Streaming via Full-Response Buffering
- **Context**: Clients send `"stream": true` on `/v1/chat/completions`; hailo-ollama cannot service this.
- **Alternatives Considered**:
  1. Redirect to `/v1/chat` streaming endpoint
  2. Implement true proxy-side chunk forwarding
  3. Buffer full response, emit as single-chunk SSE
- **Selected Approach**: Buffer the full response from hailo-ollama (non-streaming backend call), run the complete existing transformation pipeline, then emit the result as a well-formed SSE response.
- **Rationale**: The transformation pipeline (tool call rewriting, retry-on-rejection, follow-up truncation) is incompatible with real streaming — it must inspect and mutate the entire response body. Fake streaming satisfies the SSE protocol contract with no pipeline disruption.
- **Trade-offs**: The client does not receive the first token faster than non-streaming; however, HA's `extended_openai_conversation` does not visually stream tokens — it waits for the full response regardless.
- **Follow-up**: If a future hailo-ollama release supports OpenAI-compliant streaming, this layer can be replaced transparently.

### Decision: Two-Event SSE Emission Format
- **Context**: Whether to emit one or two SSE data events.
- **Alternatives Considered**:
  1. Single event: content + `finish_reason` in one chunk
  2. Two events: content event (`finish_reason: null`) + empty stop event (`finish_reason: "stop"`)
- **Selected Approach**: Two events — content delta then empty-delta stop — plus `data: [DONE]`.
- **Rationale**: Better compatibility with strict OpenAI-streaming consumers that assert `finish_reason: null` on non-final chunks.
- **Trade-offs**: Two `data:` writes vs one; negligible cost.

### Decision: Stream Flag Extraction After fix_json_control_chars
- **Context**: Where in `_forward()` to detect and strip `"stream": true`.
- **Selected Approach**: After `fix_json_control_chars` (so the body is valid JSON) and before `inject_tool_prompt`.
- **Rationale**: `fix_json_control_chars` may change byte content; parsing must happen on clean JSON. The `stream` key must be absent from the body forwarded to hailo-ollama.

## Risks & Mitigations
- **Body is not valid JSON when stream flag is extracted** — `extract_stream_flag` returns the original bytes unchanged if `json.loads` fails; `is_streaming` is `False`; the request falls through to the normal non-streaming path.
- **Client expects chunked Transfer-Encoding** — omitting `Content-Length` and writing all SSE bytes then closing is sufficient for HTTP/1.1; HA uses persistent connections but tolerates this.
- **tool_calls delta format in SSE not accepted by client** — follow-up: validate against HA's extended_openai_conversation code. Mitigation: emit complete tool_calls array in the delta, matching what HA expects in a non-streaming tool_calls response.

## References
- [OpenAI Chat Completions Streaming](https://platform.openai.com/docs/api-reference/chat/create) — SSE format specification
- [Python http.server docs](https://docs.python.org/3/library/http.server.html) — BaseHTTPRequestHandler interface
