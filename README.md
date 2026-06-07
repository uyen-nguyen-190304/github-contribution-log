# Contribution [1]: server: support control vectors

**Contribution Number:** 1

**Student:** Uyen Nguyen

**Issue:** https://github.com/ggml-org/llama.cpp/issues/6316

**Status:** Phase I — In Progress

---

## Why I Chose This Issue

This issue is labeled `good first issue` by the maintainers and has been open since March 2024 with no merged solution, making it a real gap rather than a cosmetic one. The feature — exposing control vectors through the server API — is genuinely useful: control vectors let users steer model behavior (tone, style, personality) at inference time, and right now that only works via CLI, not through the HTTP server that most production deployments use.

The scope is well-defined and has a clear reference implementation in PR #6289. The main reason it was never merged wasn't technical complexity — it was that the original author didn't write tests. That makes this an opportunity to contribute something real while building familiarity with how llama.cpp structures its server and test framework.

---

## Understanding the Issue

### Problem Description

The llama.cpp CLI (`llama-cli`) already supports control vectors via `--control-vector`, `--control-vector-scaled`, and `--control-vector-layer-range` flags. However, `llama-server` — the HTTP server used by most integrations — has no equivalent support. You cannot load or apply control vectors when running the server, either at startup or per-request.

### Expected Behavior

Users should be able to pass `--control-vector` (and related flags) to `llama-server` at startup, and have control vectors apply to completions. Ideally, the feature is also tested in the server's test framework so it doesn't regress.

### Current Behavior

Passing `--control-vector` to `llama-server` either fails silently or is not recognized. There is no way to use control vectors through the HTTP API.

### Affected Components

- `tools/server/server.cpp` — main server logic, request handling, CLI arg parsing
- `common/common.h` / `common.cpp` — `gpt_params::control_vectors` already exists here; needs to be wired through to the server
- `tests/test-server.py` (or equivalent) — server test framework where a new test scenario needs to be added

---

## Reproduction Process

### Environment Setup

> To be filled in as environment is configured.

Steps planned:
1. Clone `ggml-org/llama.cpp` and create a fork
2. Build with `cmake -B build && cmake --build build --config Release`
3. Download a small quantized model (e.g. `ggml-org/gemma-3-1b-it-GGUF`) for testing
4. Obtain or generate a control vector `.gguf` file for testing

### Steps to Reproduce

1. Build `llama-server`
2. Run: `./build/bin/llama-server -m model.gguf --control-vector myvector.gguf`
3. Observe: flag is not recognized / has no effect on completions

### Reproduction Evidence

- **Commit showing reproduction:** _To be added_
- **Screenshots/logs:** _To be added_
- **My findings:** _To be added after local setup_

---

## Solution Approach

### Analysis

The CLI tooling already has full control vector support wired through `gpt_params`. The server just never plumbed those parameters through. PR #6289 (the prior attempt) confirmed the approach is straightforward — the same pattern used for LoRA adapters applies here. The PR was not merged solely because it lacked a test case in the server test framework.

A secondary complexity is hot-swapping (changing vectors at runtime via API). PR #6289 attempted this but ran into issues with `llama_control_vector_init` being designed for one-time GPU allocation. For this contribution, the safer and more likely-to-merge approach is **startup-time loading only** first, matching the minimal requirements of issue #6316.

### Proposed Solution

Wire `gpt_params::control_vectors` through the server's initialization path (matching how LoRA is handled), and add a test scenario to the server test framework to verify the feature works and prevent regression.

### Implementation Plan

**Understand:** The server doesn't pass `--control-vector` CLI args through to the model initialization, unlike `llama-cli` which does.

**Match:** LoRA adapter loading in `server.cpp` is the closest existing pattern — it reads from `gpt_params` during startup and applies adapters to the model context. Control vectors should follow the same flow.

**Plan:**
1. Study how `--lora` is handled in `tools/server/server.cpp` as the reference pattern
2. Check `common/arg.cpp` to confirm `--control-vector` args already parse into `gpt_params::control_vectors`
3. In `server.cpp`, find where `llama_init_from_gpt_params` or equivalent is called and verify `control_vectors` is passed through
4. If not wired: add the same initialization call that `llama-cli` uses
5. Write a test scenario in the server test framework (`tests/`) that:
   - Starts the server with a `--control-vector` flag pointing to a test vector
   - Sends a completion request
   - Verifies the server starts and responds without error
6. Check against current master for merge conflicts vs. PR #6289

**Implement:** _Links to branch/commits to be added as work progresses_

**Review:**
- [ ] Follows existing code style in `server.cpp`
- [ ] No new warnings introduced
- [ ] Test added in the server test framework
- [ ] PR description references issue #6316 and prior PR #6289

**Evaluate:** Run the server test suite locally with the new test scenario passing.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Server starts successfully with a valid `--control-vector` file
- [ ] Test case 2: Server starts successfully with `--control-vector-scaled` and a scale value
- [ ] Test case 3: Server returns a completion response when a control vector is loaded (no crash/segfault)

### Integration Tests

- [ ] Server test framework scenario: load control vector at startup, verify completion endpoint responds correctly
- [ ] Verify behavior matches the equivalent `llama-cli` invocation

### Manual Testing

> To be filled in after implementation.

---

## Implementation Notes

### Week 1 Progress

> To be filled in.

### Code Changes

- **Files modified:** _To be determined — expected: `tools/server/server.cpp`, possibly `common/arg.cpp`, test file_
- **Key commits:** _To be added_
- **Approach decisions:** Starting with startup-only (no hot-swap) to keep scope tight and maximize chance of merge, per maintainer guidance in the issue thread

---

## Pull Request

**PR Link:** _To be added_

**PR Description:** _Draft below:_

> Implements `--control-vector`, `--control-vector-scaled`, and `--control-vector-layer-range` support in `llama-server`, matching the existing CLI behavior in `llama-cli`. Adds a test scenario in the server test framework.
>
> Closes #6316. References prior attempt #6289.

**Maintainer Feedback:**
> _To be filled in after PR submission_

**Status:** Not yet submitted

---

## Learnings & Reflections

> To be filled in after completion.

---

## Resources Used

- Issue #6316: https://github.com/ggml-org/llama.cpp/issues/6316
- Prior PR #6289: https://github.com/ggml-org/llama.cpp/pull/6289
- LoRA adapter implementation in `server.cpp` (reference pattern)
- llama.cpp server test framework: `tests/test-server.py`
- llama.cpp contributing guide: https://github.com/ggml-org/llama.cpp/blob/master/CONTRIBUTING.md