# Google Docs Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Google Docs datasource; the general adapter boundary is in `integration/external-services.md`.

## Overview

Google Docs are indexed by `GoogleDocDatasourceProcessor`, a subclass of the LLM-routing processor base.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/datasource/google_doc/` | Google Docs processor | `src/codemie/datasource/google_doc/google_doc_datasource_processor.py::GoogleDocDatasourceProcessor` |
| `src/codemie/datasource/base_llm_routing_processor.py` | LLM-routing processor base | `src/codemie/datasource/base_llm_routing_processor.py::BaseLLMRoutingDatasourceProcessor` |

## Google Docs

### Rules
- R1. Index Google Docs through `GoogleDocDatasourceProcessor` on the LLM-routing processor base, not in API handlers. Evidence: `src/codemie/datasource/google_doc/google_doc_datasource_processor.py::GoogleDocDatasourceProcessor`
