# Implementation Plan

## Task Overview
- **Feature**: chat-completions-streaming
- **Total**: 4 major tasks, 7 sub-tasks
- **Requirements covered**: 1.1–1.3, 2.1–2.3, 3.1–3.5, 4.1–4.4, 5.1–5.3

---

- [x] 1. Add core streaming helper functions
- [x] 1.1 (P) Implement stream flag extraction
  - Parse the request body (already clean JSON after `fix_json_control_chars`) and inspect the `"stream"` key.
  - Return the body unchanged with `is_streaming=False` when `"stream"` is absent, `false`, or the body is not valid JSON.
  - Return the body with `"stream"` removed and `is_streaming=True` when `"stream"` is `true`.
  - No other keys in the JSON object are altered.
  - Observable: calling the function with `{"model":"m","stream":true}` returns `(b'{"model":"m"}', True)`; calling with `{"model":"m"}` returns `(original_bytes, False)`.
  - _Requirements: 1.1, 1.2, 1.3, 2.1_
  - _Boundary: extract_stream_flag_

- [x] 1.2 (P) Implement SSE chunk conversion
  - Accept the fully-transformed response bytes from the pipeline and produce UTF-8 SSE bytes.
  - For content responses: emit two `data:` events — first carries `delta.role + delta.content` with `finish_reason: null`; second carries empty `delta` with `finish_reason: "stop"` — followed by `data: [DONE]`.
  - For tool-call responses (where `message.tool_calls` is set): emit first chunk with `delta.role + delta.tool_calls` and `finish_reason: null`; second chunk with empty `delta` and `finish_reason: "tool_calls"`; then `data: [DONE]`.
  - Forward `id`, `model`, and `created` from the original response into each chunk; use defaults (`"chatcmpl-stream"`, `""`, current Unix timestamp) if absent.
  - Fall back to wrapping the raw decoded text in `delta.content` if the response body cannot be parsed as JSON.
  - Observable: output starts with `b"data: "` and ends with `b"data: [DONE]\n\n"` for any well-formed input.
  - _Requirements: 3.2, 3.3, 3.4, 5.3_
  - _Boundary: response_to_sse_chunks_

- [x] 2. Implement streaming HTTP response method
- [x] 2.1 Add `_send_streaming` to the Handler class
  - Write an HTTP 200 status line and `Content-Type: text/event-stream` header.
  - Omit the `Content-Length` header entirely (SSE body length is not pre-declared).
  - Write all SSE bytes to the socket in a single call after `end_headers`.
  - Observable: a test that calls `_send_streaming(sse_bytes)` receives an HTTP response with `Content-Type: text/event-stream` and no `Content-Length` header.
  - _Requirements: 3.1, 3.5_
  - _Boundary: Handler._send_streaming_

- [x] 3. Wire streaming into the request/response pipeline
- [x] 3.1 Integrate stream detection into `_forward`
  - After `fix_json_control_chars` and before `inject_tool_prompt`, call `extract_stream_flag` and store the returned `is_streaming` boolean.
  - Pass the stream-stripped body through the rest of the existing pipeline unchanged (`inject_tool_prompt` → `sanitize_conversation_roles` → `sanitize_for_hailo` → `inject_defaults`).
  - Call `_send_to_backend` once and collect the full response — no change to backend interaction.
  - Observable: a request with `"stream": true` forwarded to the backend no longer contains the `"stream"` key; the backend returns a non-streaming JSON response.
  - _Requirements: 1.1, 2.2, 2.3_

- [x] 3.2 Integrate SSE dispatch into `_forward`
  - After all response transforms (tool-call rewrite + optional retry, or follow-up truncation) and the existing `_debug_log` call, branch on `is_streaming`.
  - When `is_streaming` is `True`: call `response_to_sse_chunks` on the transformed response, then call `_send_streaming` with the result.
  - When `is_streaming` is `False`: call `_send` as today — no change to this path.
  - The `urllib.error.HTTPError` and generic exception handlers remain unmodified; they fire before the branch and return errors identically for both paths.
  - Log `[proxy] streaming request detected on /v1/chat/completions` to stderr when the stream flag is found.
  - Observable: a `POST /v1/chat/completions` with `"stream": true` returns an HTTP response with `Content-Type: text/event-stream`; the same request without `"stream"` returns `application/json` as before.
  - _Requirements: 2.2, 2.3, 4.1, 4.2, 4.3, 4.4, 5.1, 5.2_
  - _Depends: 3.1_

- [x] 4. Validate streaming behaviour
- [x] 4.1 Verify non-streaming requests are unaffected
  - Confirm that all existing test cases in `test_proxy.py` pass without modification.
  - Confirm that requests to `/v1/chat/completions` without `"stream"` still return `application/json` responses with `Content-Length` set.
  - Confirm tool-call rewriting, retry-on-rejection, and follow-up truncation still operate correctly on non-streaming requests.
  - Observable: full existing test suite passes green after the changes are applied.
  - _Requirements: 1.2, 4.1, 4.2, 4.3, 4.4_

- [x] 4.2 Test streaming response format
  - Add unit tests for `extract_stream_flag`: cover absent key, `false`, `true`, invalid JSON, and preservation of other keys.
  - Add unit tests for `response_to_sse_chunks`: cover content response, tool-calls response, missing `id`/`model`/`created` fields, and invalid JSON input.
  - Observable: new test cases assert that the SSE output starts with `data: ` and ends with `data: [DONE]\n\n`, and that `finish_reason` is `"tool_calls"` when tool_calls are present.
  - _Requirements: 1.1, 1.2, 3.2, 3.3, 3.4, 5.3_
