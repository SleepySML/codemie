# LLM Providers

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Model provider config, LiteLLM, request tagging and speech-to-text; other external services are in `integration/external-services.md`.

## Overview

Models come from the YAML provider files selected by `MODELS_ENV`, plus LiteLLM models when LiteLLM is enabled; `LLMService` holds the catalogue. Direct provider clients receive CodeMie tagging headers built in one helper.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/service/llm_service/` | model catalogue | `src/codemie/service/llm_service/llm_service.py::LLMService` |
| `src/codemie/core/dependecies.py` | provider client factories and tagging headers | `src/codemie/core/dependecies.py::_build_codemie_tagging_headers` |
| `src/codemie/enterprise/litellm/` | LiteLLM integration | `src/codemie/rest_api/main.py::_setup_litellm_features` |
| `src/codemie/service/stt_service.py` | speech-to-text | `src/codemie/service/stt_service.py::SttService` |

## LLM providers

### Rules
- R1. Read models from the provider YAML selected by `MODELS_ENV` and from `LLMService`; do not hardcode model names in feature code. Evidence: `src/codemie/service/llm_service/llm_service.py::LLMService`
- R2. Gate LiteLLM behaviour on `is_litellm_enabled()` and reach it through the enterprise provider registry. Evidence: `src/codemie/rest_api/main.py::_setup_litellm_features`
- R3. Add CodeMie tagging headers in `_build_codemie_tagging_headers` for direct provider clients only (LiteLLM traffic is tagged with `x-litellm-tags`), carry no personal data, and let configured `client_headers` win on collision. Evidence: `src/codemie/core/dependecies.py::_build_codemie_tagging_headers`
- R4. Build speech-to-text calls through `SttService`; a model forwards only the request headers its `stt_config` lists, minus the blocklist. Evidence: `src/codemie/service/stt_service.py::select_forward_headers`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A `bedrock_converse` model is called through Azure OpenAI | the LiteLLM provider mapping lacked the `bedrock_converse` entry (EPMCDME-14015) | add the provider to the mapping and to its test | `src/codemie/enterprise/litellm/models.py::bedrock_converse` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a tagging header | extend `_build_codemie_tagging_headers` | `src/codemie/core/dependecies.py::_build_codemie_tagging_headers` |
| Choose the forwarded STT request headers | list them in the model's `stt_config` | `src/codemie/service/stt_service.py::select_forward_headers` |
| Check whether LiteLLM is on at startup | read `_setup_litellm_features` | `src/codemie/rest_api/main.py::_setup_litellm_features` |
