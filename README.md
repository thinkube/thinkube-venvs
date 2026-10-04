# thinkube-venvs

Release files of the two Python environments that Thinkube notebook
servers use as Jupyter kernels: `fine-tuning` and `agent-dev`.

## What it does

The repository holds no code. Its GitHub releases hold four tarballs, one
per environment and architecture:

- `arm64-fine-tuning.tar.gz`, `amd64-fine-tuning.tar.gz`
- `arm64-agent-dev.tar.gz`, `amd64-agent-dev.tar.gz`

Each tarball is a Python virtual environment created with
`--system-site-packages`, so it uses the PyTorch of the `tk-jupyter-base`
image and does not carry its own. Each one also carries its Jupyter kernel
definition, named `<environment> (<architecture>)`, for example
`fine-tuning (arm64)`.

## How it reaches a user

These are release files, built and downloaded by the core platform in
[thinkube](https://github.com/thinkube/thinkube):

- **Built** by the playbook
  `ansible/40_thinkube/core/jupyterhub/99_build_venvs.yaml`. It builds the
  environments on a GPU node of each architecture and uploads the tarballs
  to the release named by `venvs_version`.
- **Downloaded** by every notebook server. The `setup-venvs` init
  container in `core/jupyterhub/templates/jupyterhub-values.yaml.j2`
  downloads the two tarballs for the node's architecture from the release
  named by `jupyter_venvs_version` (today `v0.1.0`), extracts them to
  `/home/thinkube/venvs/<arch>/` on the node's local disk (a hostPath
  volume), and registers each as a Jupyter kernel. A `.version` file skips
  the download when that version is already on the node. A failed
  download stops the server start.

The environments are not installed on their own. In thinkube-control the
two are templates: they are not rebuilt there, and a custom environment
can start from one of them.

## Standalone use

Standalone use is not supported. The environments are built and tested
only as part of the Thinkube platform. The licence lets you use them
anywhere, but issues and questions about using an environment outside
Thinkube are not answered.

## Contents of the two environments

The package lists are in `ansible/40_thinkube/core/jupyterhub/venv-packages.txt`
in thinkube. The build (`core/jupyterhub/scripts/build-venvs.sh`) reads
that file. `[base]` goes into both environments; `[fine-tuning]` and
`[agent-dev]` are added on top.

Both environments inherit PyTorch (and torchvision) from the
`tk-jupyter-base` image. The build fails if an environment installs its
own torch, or carries a compiled extension that does not load against the
image's torch.

### Both environments (`[base]`)

- ML and data: `transformers`, `kernels`, `datasets==4.3.0`, `accelerate`,
  `nvidia-modelopt`, `pandas`, `scikit-learn`, `sentence-transformers`,
  `spacy`
- Plots and widgets: `matplotlib`, `seaborn`, `plotly`, `ipywidgets`,
  `jupyterlab-widgets`, `tqdm`, `Pillow`, `opencv-python`
- Platform service clients: `psycopg2-binary`, `redis`, `qdrant-client`,
  `langchain-qdrant`, `opensearch-py`, `mlflow`, `boto3`,
  `clickhouse-connect`, `chromadb`, `nats-py`, `weaviate-client`,
  `kubernetes==36.0.3`, `PyGithub`, `hera-workflows`, `argilla`,
  `cvat-sdk`, `langfuse`
- LLM clients: `litellm`, `openai`, `anthropic`, `tk-llm[openai]`,
  `claude-agent-sdk`, `openai-harmony`
- General: `ipykernel`, `arxiv`, `python-dotenv`, `requests`, `httpx`,
  `pydantic`, `sqlalchemy`, `alembic`, `grpcio`, `grpcio-tools`, `gql`,
  `websockets`

### `fine-tuning`

`[base]`, plus:

- `bitsandbytes`, `peft`, `trl`, `tyro`, `hf_transfer`, `sentencepiece`,
  `protobuf`, `openpyxl`, `python-constraint`, `flash-linear-attention`,
  `torchao>=0.16`
- `causal-conv1d`, built from source against the image's torch, for CUDA
  architectures 8.0, 8.6, 8.9, 9.0 and 12.0
- Unsloth and unsloth-zoo, installed from their git repositories with
  `--no-deps`

### `agent-dev`

`[base]`, plus:

- `langchain==1.4.0`, `langchain-core==1.6.3`, `langchain-community==0.4.2`,
  `langchain-openai==1.6.2`, `langgraph==1.2.11`
- `ag2[openai]==0.10.2`, `openai-agents==0.20.0`, `crewai==1.6.1`,
  `crewai-tools==1.6.1`
- `faiss-cpu`, `tiktoken`, `opentelemetry-sdk`,
  `opentelemetry-exporter-otlp`, `opentelemetry-api`
- `openlit`, installed with `--no-deps`

The reasons for each pin and each install flag are written as comments in
`venv-packages.txt` and `build-venvs.sh`.

## Working on it

To change an environment, change `venv-packages.txt` or `build-venvs.sh`
in thinkube, then run the build playbook:

```bash
./scripts/tk_ansible ansible/40_thinkube/core/jupyterhub/99_build_venvs.yaml
```

The playbook needs:

- the `tk-jupyter-base` image in Harbor, for both architectures
- `kubectl` access to the cluster
- a GPU node of each architecture (arm64 and amd64)
- the `gh` CLI, logged in with write access to this repository

It builds one architecture after the other, deletes the release named by
`venvs_version` if it exists, creates it again and uploads the four
tarballs. Notebook servers download a new version only when
`jupyter_venvs_version` names it.

## License

Apache License 2.0. See [LICENSE](LICENSE).
