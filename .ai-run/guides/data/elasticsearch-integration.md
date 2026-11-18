# Elasticsearch Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Elasticsearch repositories and index setup; search size limits are in `data/database-optimization.md`.

## Overview

Elasticsearch documents are read and written through `BaseElasticRepository` subclasses or `BaseModelWithElasticSupport` models. Indexes the app owns are created at startup, with failures logged as warnings.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/repository/base_elastic_repository.py` | repository base over the ES client | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository` |
| `src/codemie/rest_api/models/base.py` | Elasticsearch-backed model base | `src/codemie/rest_api/models/base.py::BaseModelWithElasticSupport` |
| `src/codemie/clients/elasticsearch.py` | client construction | `src/codemie/clients/elasticsearch.py::ElasticSearchClient` |

## Elasticsearch access

### Rules
- R1. Build Elasticsearch queries inside a `BaseElasticRepository` subclass or an Elasticsearch model and convert hits through `to_entity`; routers do not build query bodies. Evidence: `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.to_entity`
- R2. Create an index the app owns at startup, next to the conversation analytics index, and log a failure with context. Evidence: `src/codemie/rest_api/main.py::_setup_conversation_analysis_scheduler`

## Examples

### Searching an index and converting hits

Source: `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.get_all`

```python
    def get_all(self, query: Optional[dict] = None, limit: Optional[int] = None) -> list[AbstractElasticModel]:
        if query is None:
            query = {"match_all": {}}

        size = limit if limit is not None else 10000
        repos_result = self._elastic_client.search(index=self._index_name, query=query, size=size)

        result = []

        for hit in repos_result["hits"]["hits"]:
            try:
                entity = self.to_entity(hit["_source"])
                result.append(entity)
            except RequestValidationError:
                logger.warning(f"Invalid document found in {self._index_name} index: {hit['_source']}")
                continue

        return result
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A list from `get_all` silently misses documents | `get_all` skips a hit that fails `to_entity` validation with a warning, and caps results at 10000 when no limit is given | pass an explicit `limit` and fix the documents the warning names | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.get_all` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Search by name | call `search_by_name` | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.search_by_name` |
| Create an index at startup | add it next to the conversation analytics index setup | `src/codemie/rest_api/main.py::_setup_conversation_analysis_scheduler` |
| Map ES failures to HTTP | let `elastic_exception_handler` return 503 | `src/codemie/rest_api/main.py::elastic_exception_handler` |
