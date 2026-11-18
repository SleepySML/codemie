# Database Optimization

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Pagination, batching and search size limits; sessions and transactions are in `data/database-patterns.md`.

## Overview

List endpoints and repositories bound their reads: Elasticsearch reads pass `size`, and SQL list helpers page with `page` and `per_page`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/repository/base_elastic_repository.py` | search size handling | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.get_all` |
| `src/codemie/repository/ai_kata_repository.py` | paged SQL reads | `src/codemie/repository/ai_kata_repository.py::get_all_published` |

## Bounded reads

### Rules
- R1. Page list reads (`page`, `per_page`) and read many rows with `get_by_ids` instead of a loop of single reads. Evidence: `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.get_by_ids`
- R2. Pass an explicit `limit` to `BaseElasticRepository.get_all`; without it the search size is 10000 and hits that fail `to_entity` are skipped with a warning. Evidence: `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.get_all`

### Read first
| When you are | Exemplar | Why this one | Tests |
|---|---|---|---|
| reading many rows by id on an async session | `src/codemie/repository/user_repository.py::UserRepository.aget_users_by_ids` | one `IN` read per `MAX_POSTGRES_QUERY_ARGUMENTS` chunk, returns an id-keyed dict, skips non-UUID ids | `tests/codemie/repository/test_user_repository_crud.py::TestUserRepositoryAsyncGetUsersByIds` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Bound an ES read | pass `limit` to `get_all` | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.get_all` |
| Page a SQL list | take `page` and `per_page` like the kata repository | `src/codemie/repository/ai_kata_repository.py::get_all_published` |
| Read many rows by id | use `get_by_ids` instead of a loop of `get_by_id` | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.get_by_ids` |
