# Cloud Integrations

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Cloud toolkits and file storage repositories; the general adapter boundary is in `integration/external-services.md`.

## Overview

Cloud operations are generic per-provider tools under `src/codemie_tools/cloud/` (AWS, Azure, GCP, Kubernetes) grouped by one `CloudToolkit`; file storage on the same clouds is a repository chosen by `FileRepositoryFactory`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie_tools/cloud/` | cloud tools and toolkit | `src/codemie_tools/cloud/toolkit.py::CloudToolkit` |
| `src/codemie/repository/` | cloud file repositories | `src/codemie/repository/aws_file_repository.py::AWSFileRepository` |

## Cloud

### Rules
- R1. Extend the provider package under `src/codemie_tools/cloud/` (aws, azure, gcp, kubernetes) for cloud operations instead of a new client path. Evidence: `src/codemie_tools/cloud/toolkit.py::CloudToolkit`
- R2. Store files through the repository `FileRepositoryFactory` returns, not service-level branches per cloud. Evidence: `src/codemie/repository/repository_factory.py::FileRepositoryFactory._get_repository`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change an AWS tool | edit the generic AWS tool | `src/codemie_tools/cloud/aws/tools.py::GenericAWSTool` |
| Change a Kubernetes tool | edit the generic Kubernetes tool | `src/codemie_tools/cloud/kubernetes/tools.py::GenericKubernetesTool` |
| Store files in S3 | set the storage type to `aws`; the factory builds `AWSFileRepository` | `src/codemie/repository/repository_factory.py::FileRepositoryFactory._get_repository` |
