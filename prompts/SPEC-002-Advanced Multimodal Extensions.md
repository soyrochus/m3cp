# SPEC-002 — Advanced Multimodal Extensions (Non-Realtime)

Status: Draft
Owner: (you)
Last updated: 2026-01-15
Depends on: SPEC-001 (Base generation)
Scope: Additional multimodal MCP tools extending vision, audio, and deterministic cross-modal composition using the OpenAI API.

---

## 0. Outcome

Extend the existing Multimodal MCP Server with **higher-order multimodal capabilities** that remain:

* file-first
* deterministic in side effects
* OpenAI-only
* suitable for agentic clients (IDEs, orchestrators, pipelines)

This SPEC introduces **six new MCP tools**:

Vision:

1. `image_edit`
2. `image_extract`
3. `image_to_spec`

Audio:
4. `audio_analyze`
5. `audio_transform`

Cross-modal:
6. `multimodal_chain`

No realtime, streaming, or video functionality is specified or implemented here.

---

## 1. Non-goals (explicit)

Do not implement or specify:

* Realtime or streaming APIs (voice sessions, low-latency loops).
* Video ingestion or analysis.
* Autonomous agents, planning, retries beyond a single tool call.
* Implicit pipelines or hidden orchestration.
* Non-OpenAI providers.

The server remains a **deterministic tool host**, not an agent.

---

## 2. Design principles (additive)

In addition to SPEC-001 principles:

* Every tool must map cleanly to **one OpenAI API call**, except `multimodal_chain`, which is a **deterministic sequence** of declared steps.
* No tool may invent intermediate paths, filenames, or resources.
* Cross-modal composition must be **explicit, auditable, and reproducible**.
* “Analysis” tools return data; “generation” tools write files.

---

## 3. Tool surface (additions)

The following tools are added to the MCP surface. Names are stable API.

### Vision

* `image_edit`
* `image_extract`
* `image_to_spec`

### Audio

* `audio_analyze`
* `audio_transform`

### Cross-modal

* `multimodal_chain`

Each tool follows the same result envelope defined in SPEC-001:

```
{
  ok: bool,
  outputs?: [...],
  metadata?: {...},
  warnings?: [...],
  error?: { code, message }
}
```

---

## 4. Vision extensions

### 4.1 `image_edit`

**Purpose**
Edit or inpaint an existing image using a natural-language prompt and optional mask.

**Inputs**

* `image_ref: str` (required)
* `prompt: str` (required)
* `mask_ref: str` (optional) — transparent regions indicate editable areas
* `output_ref: str` (required)
* `format: str` (optional) — png | jpeg | webp
* `size: str` (optional) — must match source image if required by model
* `overwrite: bool` (optional, default false)

**Behavior**

* Read source image (and mask if provided).
* Call OpenAI Images edit endpoint.
* Write result to `output_ref`.
* Return output metadata including sha256.

**Notes**

* If the underlying model does not support masks, reject with `UNSUPPORTED_FORMAT`.
* Mask and image dimensions must match exactly; otherwise reject.

---

### 4.2 `image_extract` (Structured vision / OCR++)

**Purpose**
Extract structured information from an image (documents, tables, UI screenshots) using schema-constrained vision output.

**Inputs**

* `image_ref: str` (required)
* `instruction: str` (required) — e.g. “Extract table rows”, “Read form fields”
* `json_schema: dict` (required)
* `language: str` (optional)
* `max_output_tokens: int` (optional)

**Behavior**

* Read image bytes.
* Call OpenAI Responses API with vision input.
* Require schema-valid JSON output.
* Validate JSON strictly against provided schema.
* Return JSON inline (never written to disk).

**Failure modes**

* Invalid or non-conformant JSON → `INVALID_ARGUMENT`
* Partial extraction → return best effort with warning, not failure

**Rationale**

This formalizes OCR-like usage without introducing a separate OCR engine and avoids free-text post-processing in clients.

---

### 4.3 `image_to_spec`

**Purpose**
Transform diagrams or visual representations into formal textual artifacts.

Typical inputs:

* Architecture diagrams
* UI wireframes
* Sequence diagrams
* Whiteboard photos

**Inputs**

* `image_ref: str` (required)
* `target_format: str` (required)
  Allowed values (extensible):

  * `mermaid`
  * `plantuml`
  * `openapi`
  * `c4`
  * `markdown`
* `instruction: str` (optional) — refinement or constraints
* `output_ref: str` (optional)
* `overwrite: bool` (optional)

**Behavior**

* Analyze image via Responses API.
* Generate textual artifact in requested format.
* If `output_ref` provided, write to file; otherwise return inline.

**Constraints**

* The tool does not validate semantic correctness of the produced spec.
* Syntax validity is best-effort; return warnings if uncertain.

---

## 5. Audio extensions

### 5.1 `audio_analyze`

**Purpose**
Analyze audio content beyond transcription.

Supported analyses include:

* sentiment
* tone
* speaker dynamics
* acoustic events (laughter, silence, emphasis)
* meeting quality signals

**Inputs**

* `audio_ref: str` (required)
* `instruction: str` (required)
* `response_format: str` (optional) — text | json
* `json_schema: dict` (required if response_format=json)

**Behavior**

* Read audio bytes.
* Call OpenAI audio-capable reasoning model.
* Return analysis inline.
* Enforce schema if JSON requested.

**Non-goal**

* This tool does not output audio or transcripts.

---

### 5.2 `audio_transform` (speech-to-speech)

**Purpose**
Transform spoken audio into new spoken audio based on an instruction.

Examples:

* spoken language translation
* tone change
* summarization spoken aloud
* role-play or style shift

**Inputs**

* `audio_ref: str` (required)
* `instruction: str` (required)
* `output_ref: str` (required)
* `voice: str` (optional)
* `format: str` (optional) — mp3 | wav | opus
* `overwrite: bool` (optional)

**Behavior**

* Read input audio.
* Call OpenAI speech-to-speech capable endpoint.
* Write transformed audio to output.
* Return metadata + sha256.

**Constraints**

* Voice preservation is best-effort.
* If speech-to-speech is unavailable in the configured model, reject with `UNSUPPORTED_FORMAT`.

---

## 6. Cross-modal composition

### 6.1 `multimodal_chain`

**Purpose**
Execute a deterministic, explicitly declared sequence of multimodal operations.

This is **not an agent** and contains no dynamic planning.

**Inputs**

* `steps: list` (required)

Each step:

```
{
  tool: str,
  args: dict,
  outputs_as: str | null
}
```

* `final_output_ref: str` (optional)
* `overwrite: bool` (optional)

**Execution model**

* Steps execute sequentially.
* Each step may reference previous outputs via symbolic names.
* No branching, no loops, no conditionals.

**Example**

1. image_analyze → text
2. audio_tts → speech
3. write final audio

**Failure behavior**

* On first failure, stop execution.
* Return partial outputs and error.

**Security**

* Each step is validated independently.
* All file access rules apply per step.

**Rationale**

This enables reproducible multimodal pipelines while avoiding agent semantics or hidden control flow.

---

## 7. Configuration additions

Optional environment variables:

* `OPENAI_MODEL_IMAGE_EDIT`
* `OPENAI_MODEL_AUDIO_ANALYZE`
* `OPENAI_MODEL_AUDIO_TRANSFORM`

Defaults must fall back to sensible OpenAI models if unset.

---

## 8. Error codes (additions)

Add to the stable error set:

* `SCHEMA_VALIDATION_FAILED`
* `CHAIN_STEP_FAILED`
* `UNSUPPORTED_TRANSFORMATION`

All existing error-handling rules from SPEC-001 apply.

---

## 9. Testing requirements (additive)

* Unit tests for:

  * schema enforcement (`image_extract`, `audio_analyze`)
  * step resolution and failure propagation (`multimodal_chain`)
* Mock OpenAI client responses for all new tools.
* At least one opt-in live test per modality extension.



## 10. Documentation model

The project documentation is split into two layers:

### README.md
Acts as an entry point. It explains what the server is, what tools it exposes, how to run it, and where to find deeper documentation.
It does not attempt to teach usage patterns beyond minimal examples.

### docs/MCP3-Manual.md (MCP³ Manual)
A technical, user-facing manual describing how to use the Multimodal MCP Server in practice.
It implement the techical spec as contained in this document  and adds to this, complements, by explaining tool semantics, constraints, and common usage patterns from a client perspective (IDE agents, orchestrators, pipelines).

---

## 11. Acceptance criteria

This SPEC is satisfied when:

* All six tools are registered and callable via MCP.
* No implicit file access occurs.
* Schema enforcement is strict and predictable.
* Multimodal chains are reproducible and auditable.
* No realtime, streaming, or video functionality exists in code or spec.

