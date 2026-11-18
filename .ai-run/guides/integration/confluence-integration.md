# Confluence Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Confluence datasource processing; Jira is in `integration/jira-integration.md`.

## Overview

`ConfluenceDatasourceProcessor` indexes Confluence through `ConfluenceDatasourceLoader`, a subclass of the LangChain loader that overrides CQL search pagination.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/datasource/confluence_datasource_processor.py` | Confluence processor | `src/codemie/datasource/confluence_datasource_processor.py::ConfluenceDatasourceProcessor` |
| `src/codemie/datasource/loader/confluence_loader.py` | Confluence loader overrides | `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader` |

## Confluence

### Rules
- R1. Change Confluence ingestion in `ConfluenceDatasourceProcessor` and its loader; keep the CQL search following the `_links.next` URL (EPMCDME-14246 fixed an endless loop there). Evidence: `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader._search_content_by_cql`

## Examples

### CQL search that follows the next link

Source: `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader._search_content_by_cql`

```python
    def _search_content_by_cql(
        self,
        cql: str,
        include_archived_spaces: Optional[bool] = None,
        next_url: str = "",
        **kwargs: Any,
    ) -> tuple[List[dict], str]:
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Confluence indexing loops without end | the CQL search did not follow the `_links.next` URL between pages (EPMCDME-14246) | keep `_search_content_by_cql` returning the next link and requesting it | `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader._search_content_by_cql` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change remote page statistics | edit `fetch_remote_stats` | `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader.fetch_remote_stats` |
| Change how pages are loaded | edit `lazy_load` | `src/codemie/datasource/loader/confluence_loader.py::ConfluenceDatasourceLoader.lazy_load` |
| Change the Confluence loader setup | edit `_init_loader` | `src/codemie/datasource/confluence_datasource_processor.py::ConfluenceDatasourceProcessor._init_loader` |
