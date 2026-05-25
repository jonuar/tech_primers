# Tech Cheatsheets

Quick reference sheets for technologies I work with regularly. Each sheet covers: mental model, install/setup, core concepts with annotated code, most-used patterns, and gotchas.

---

## Index

### Frameworks
| Sheet | Focus |
|---|---|
| [React.js](frameworks/reactjs.md) | Hooks, state, component patterns |
| [LangChain](frameworks/langchain.md) | LCEL, RAG chains, retrievers, agents, memory |
| [FastAPI + Pydantic](frameworks/fastapi.md) | Routing, DI, Pydantic models, lifespan, async, routers |

### AI / Vector
| Sheet | Focus |
|---|---|
| [pgvector](ai/pgvector.md) | Vector type, distance operators, HNSW/IVFFlat, hybrid search, LangChain |

### Cloud
| Sheet | Focus |
|---|---|
| [AWS](cloud/aws.md) | S3, Lambda, API Gateway, ECS, ECR, CloudWatch, SageMaker |
| [GCP Associate Cert](cloud/gcp-associate.md) | gcloud CLI, compute, storage, networking, IAM, exam tips |

### Infra
| Sheet | Focus |
|---|---|
| [Kubernetes](infra/kubernetes.md) | Pods, Deployments, Services, Ingress, ConfigMaps, HPA |
| [Terraform](infra/terraform.md) | Providers, resources, variables, modules, state, for_each |

### Languages
| Sheet | Focus |
|---|---|
| [Rust](languages/rust.md) | Ownership, borrowing, traits, enums, async, serde |

### ML / AI
| Sheet | Focus |
|---|---|
| [PyTorch](ml/pytorch.md) | Tensors, autograd, training loop, DataLoader, save/load |

### Testing
| Sheet | Focus |
|---|---|
| [Pytest](testing/pytest.md) | Fixtures, parametrize, mocking, FastAPI testing, coverage |

### Tools
| Sheet | Focus |
|---|---|
| [Neovim](tools/nvim.md) | Modes, motions, text objects, macros, Lua config, plugins |

---

## Template

Each cheatsheet follows this structure:

1. **Mental Model** — one paragraph, the core idea
2. **Install & Minimal Setup** — copy-paste ready
3. **Core Concepts** — the 5–10 things you actually need to understand
4. **Most-Used Patterns** — annotated code snippets
5. **Gotchas** — things that waste an hour if you don't know them
6. **Quick Links** — official docs, best resources

---

## Backlog

- `frameworks/langgraph.md`
- `ml/mlflow.md`
- `ml/sagemaker.md`
- `ai/chromadb.md`
- `ai/qdrant.md`
- `ai/ollama.md`
- `ai/ragas.md`
- `infra/docker.md`
- `languages/go.md`
