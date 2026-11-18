# Repository Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Repository boundary and storage factories; SQL sessions and models are in `data/database-patterns.md`.

## Overview

Repositories under `src/codemie/repository/` own storage access: Elasticsearch repositories subclass `BaseElasticRepository`, general file storage is chosen by `FileRepositoryFactory` from `FILES_STORAGE_TYPE`, and the filesystem is the default; hosted application storage is opened separately by `open_hosted_storage`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/repository/base_elastic_repository.py` | Elasticsearch repository base | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository` |
| `src/codemie/repository/repository_factory.py` | file storage selection | `src/codemie/repository/repository_factory.py::FileRepositoryFactory` |
| `src/codemie/repository/cli_analytics/` | CLI analytics storage ports and PostgreSQL implementation | `src/codemie/repository/cli_analytics/ports.py::IngestGate` |
| `src/codemie/repository/hosted_applications/` | hosted application storage (private volume or dedicated bucket) | `src/codemie/repository/hosted_applications/factory.py::open_hosted_storage` |

## Repositories

### Rules
- R1. Extend the matching repository for storage access and return project entities or models, not provider responses. Evidence: `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository`
- R2. Get file storage from `FileRepositoryFactory.get_current_repository` (chosen by `FILES_STORAGE_TYPE`, filesystem by default) instead of branching on the provider in a service. Evidence: `src/codemie/repository/repository_factory.py::FileRepositoryFactory.get_current_repository`
- R3. Open hosted application storage with `open_hosted_storage` (filesystem, aws or gcp from `FILES_STORAGE_TYPE`; private volume `HOSTED_APPS_STORAGE_DIR` or dedicated bucket `HOSTED_APPS_STORAGE_BUCKET`), not `FileRepositoryFactory`; do not route hosted app bytes through the general file repositories. Evidence: `src/codemie/repository/hosted_applications/factory.py::open_hosted_storage`

## Examples

### Selecting a file repository from config

Source: `src/codemie/repository/repository_factory.py::FileRepositoryFactory._get_repository`

```python
    def _get_repository(cls, storage_type: FileStorageType):
        """Private method to instantiate and return a file repository object based on
        the specified storage type.

        :param storage_type: An instance of `FileStorageType` indicating the desired storage type.
        :return: An instance of a file repository corresponding to the provided `storage_type`.
        :raises UnsupportedFileSystemException: If an unsupported storage type (e.g., `AWS`) is provided.
        """
        logger.debug(f"Creating file repository for storage type: {storage_type.value}")
        if storage_type == FileStorageType.GCP:
            return GCPFileRepository()
        elif storage_type == FileStorageType.AZURE:
            return AzureFileRepository(
                connection_string=config.AZURE_STORAGE_CONNECTION_STRING,
                storage_account_name=config.AZURE_STORAGE_ACCOUNT_NAME,
            )
        elif storage_type == FileStorageType.FILE_SYSTEM:
            return FileSystemRepository()
        elif storage_type == FileStorageType.AWS:
            return AWSFileRepository(region_name=config.AWS_S3_REGION, root_bucket=config.AWS_S3_BUCKET_NAME)
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An Azure container name with underscores is rejected | Azure container names forbid underscores | build container names with `FileRepositoryFactory.build_subdir`, which swaps them for hyphens on Azure | `src/codemie/repository/repository_factory.py::FileRepositoryFactory.build_subdir` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Get the configured file repository | call `FileRepositoryFactory.get_current_repository` | `src/codemie/repository/repository_factory.py::FileRepositoryFactory.get_current_repository` |
| Add an Elasticsearch repository | subclass `BaseElasticRepository` and implement `to_entity` | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository.to_entity` |
| Add a storage backend | add a `FileStorageType` member and a branch in `_get_repository` | `src/codemie/repository/repository_factory.py::FileStorageType` |
