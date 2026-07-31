---
title: 'Local-Ollama — A Self-Hosted LLM Inference Pipeline on AWS'
description: 'A production-ready, cost-optimised architecture for running your own LLM inference on AWS — OpenAI-compatible API, multi-backend routing, cost controls, and analytics.'
pubDate: 'Jul 31 2026'
tags: ['ai', 'llm', 'aws', 'self-hosting', 'ollama', 'infrastructure']
---

As organisations adopt AI at scale, one question keeps coming up: **should we run inference in the cloud on someone else's API, or bring the models in-house?**

For teams that care about data residency, cost predictability, and fine-grained control, self-hosting large language models is increasingly compelling. That's exactly the problem I set out to solve with **Local-Ollama** — a production-ready, cost-optimised architecture for running self-hosted LLM inference on AWS.

## What is Local-Ollama?

Local-Ollama is an [open-source project](https://github.com/nishantdave/Local-Ollama) that bundles everything you need to run LLMs on your own infrastructure:

- **OpenAI-compatible API gateway** — a drop-in replacement for the OpenAI SDK, LangChain, or any client using the chat/completions or embeddings format
- **Multi-backend routing** — Ollama (default), llama.cpp, and vLLM, selected via prefix-based model routing
- **Web UI** — a dashboard with model management, a chat playground, and analytics
- **Cost controls** — per-API-key monthly spending caps
- **Response caching** — Redis-backed SHA-256 cache for deterministic requests
- **Rate limiting, request logging, and analytics** — production-ready governance out of the box

## The architecture

The system follows a clean, layered design:

```
Client ──▶ ALB ──▶ ECS Cluster (Graviton3)
                  │
                  ├── API Gateway (FastAPI)
                  │     CORS → Rate Limiter → Auth → Cost Limit → Cache → Logging
                  │
                  └── Ollama / llama.cpp / vLLM
                        │
              Redis · EFS · Secrets Manager · CloudWatch
```

Every request passes through a middleware stack that enforces rate limits, validates API keys, tracks monthly spend, and serves cached responses — so the platform behaves more like a managed service than a lab experiment.

## Why the cost story matters

Running LLMs at scale on standard x86 on-demand instances gets expensive quickly. The architecture attacks that directly:

- **Graviton3 ARM64 Spot instances** — up to 90% cheaper than on-demand x86
- **Shared model storage on EFS** — models persist across task restarts and instance replacements, so you don't re-download weights
- **A realistic POC footprint** — a single c7g.xlarge Spot instance plus supporting services lands around **~$95/month**, versus comparable managed options that scale with every token

## A scaling path

The repo documents a clear evolution:

1. **POC** — a single Graviton3 Spot instance with 1–2 small models (~$95/month)
2. **Growth** — 2–4 instances with auto-scaling on CPU load and a shared model pool via EFS
3. **Production** — reserved capacity, GPU instances running vLLM, multi-AZ, HTTPS + WAF, and Cognito auth

## Getting started

One-click scripts handle the heavy lifting:

- **Local**: `deploy-local.bat` — checks Docker, builds images, starts services, and prints endpoints
- **AWS**: `deploy-aws.bat` — bootstraps CDK and deploys all stacks
- **Teardown**: `teardown-aws.bat` — destroys everything and stops billing

And because it's OpenAI-compatible, existing clients just work:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",
)

response = client.chat.completions.create(
    model="llama3.2:1b",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(response.choices[0].message.content)
```

## Why I built it

Self-hosting LLMs is about more than saving money. It's about **control** — over your models, your data, your costs, and your compliance posture. This project is my take on how an engineering team should think about that: a complete, well-governed platform rather than a single Docker container.

If you're exploring self-hosted inference, I'd love to hear your experiences. The project is open source under the MIT license — check out the [repository](https://github.com/nishantdave/Local-Ollama), open an issue, or contribute.

Let's keep innovating.
