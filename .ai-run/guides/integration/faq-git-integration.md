# FAQ Git Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Git FAQ datasource parser, indexing, routing and contracts; other datasources are in `integration/external-services.md`.

## Overview

`GitFaqDatasourceProcessor` indexes FAQ articles from a Git repository: it parses the FAQ documents, joins markdown chunks in windows and assigns each chunk its title and header.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/datasource/git_faq/` | FAQ processor | `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor` |

## Git FAQ datasource

### Rules
- R1. Keep Git FAQ a knowledge-base type indexed as embedded, header-aware chunks through `BaseDatasourceProcessor`; it registers no entry in `_LLM_ROUTING_PROCESSOR_FACTORIES` and stores no TOC (whole-document LLM routing was superseded by the 2026-09-01 retrieval decision). Evidence: `src/codemie/agents/tools/kb/search_kb.py::_LLM_ROUTING_PROCESSOR_FACTORIES`
- R2. Parse one article per Markdown file in `GitFaqLoader`: title from frontmatter `title`, then the first `#` heading, then the file name; frontmatter only at line 0. Evidence: `src/codemie/datasource/loader/git_faq_loader.py::GitFaqLoader`
- R3. Split an article at H1-H3 headings and join neighbouring sections in windows before the standard chunk splitter, so retrieval returns a section rather than the whole article. Evidence: `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor.join_markdown_chunks_by_window`
- R4. Change FAQ REST behaviour in the `kb_git_faq` router and FAQ indexing in the processor; the loader owns cloning and file selection. Evidence: `src/codemie/rest_api/routers/index/kb_git_faq.py::full_reindex`
- R5. Reject resume and incremental reindex for Git FAQ (the processor raises `NotImplementedError` and the stale-datasource watchdog skips types in `UNSUPPORTED_RESUME_TYPES`); a full `reprocess()` is the reindex path. Evidence: `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor._on_process_start`
- R6. Require `full_reindex=true` on a PUT that changes a FAQ source field or the project; without it the router returns 422. Evidence: `src/codemie/rest_api/routers/index/kb_git_faq.py::full_reindex`
- R7. Keep Git FAQ out of the code index types (`_CODE_INDEX_TYPES` holds `code`, `summary`, `chunk-summary`) and out of the LLM-routing factories; add a FAQ-specific branch instead. Evidence: `src/codemie/service/stale_datasource/stale_datasource_service.py::_CODE_INDEX_TYPES`
- R8. Treat the processor, loader, parser, REST and trigger tests as the executable FAQ contract; the retrieval decision is in docs/2026-09-01-faq-retrieval-strategy-comparison.md. Evidence: `tests/codemie/datasource/git_faq/test_git_faq_datasource_processor.py::TestGitFaqDatasourceProcessor`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change FAQ parsing | edit `_parse_faq_docs` | `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor._parse_faq_docs` |
| Change chunk windowing | edit `join_markdown_chunks_by_window` | `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor.join_markdown_chunks_by_window` |
| Change the FAQ index setup | edit `_init_index` | `src/codemie/datasource/git_faq/git_faq_datasource_processor.py::GitFaqDatasourceProcessor._init_index` |

## Notes

The Git FAQ datasource loads every Markdown article in a Git repository (optionally narrowed by
`files_filter`) and indexes embedded, header-aware chunks. Its processing and retrieval model follows
the Confluence knowledge-base flow rather than the Google Docs whole-document LLM-routing flow.
The selected design is Option A in
`docs/2026-09-01-faq-retrieval-strategy-comparison.md`: Confluence-style chunking with standard hybrid
knowledge-base retrieval.

## Verification

The processor, loader, parser, REST, and trigger tests under `tests/codemie/` define the executable
contract. The retrieval decision and live comparison are recorded in
