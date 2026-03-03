# Why I Treat LLMs Like Infrastructure, Not Magic

## Overview

A blog post exploring the architectural pattern of treating LLM providers as pluggable infrastructure backends rather than hardcoded dependencies. Includes practical examples from production systems.

---

## Why I Treat LLMs Like Infrastructure, Not Magic

There's a pattern I see repeatedly in AI-integrated codebases: the LLM provider is hardcoded at the call site.

```python
client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
response = client.messages.create(model="claude-3-5-sonnet", ...)
```

It works. For a demo, it's fine. But it's the same mistake we made with databases in the early web era — coupling business logic to a specific vendor's interface.

### The Mental Model Shift

Infrastructure doesn't care about its implementation. Your application code doesn't call MySQL-specific functions — it calls SQL. Your cache layer doesn't expose Redis internals — it exposes a get/set interface.

LLMs deserve the same treatment. Model providers are execution backends. Your business logic should speak to an interface, not a vendor.

### What This Looks Like in Practice

In [project-recon](https://github.com/yourhandle/project-recon), a freelance project discovery pipeline, the LLM layer is three files:

**base.py** — the interface:
```python
class LLMProvider(ABC):
    @abstractmethod
    def generate(self, prompt: str, system_prompt: str = None) -> str: ...

    @abstractmethod
    def generate_with_history(self, messages: list, system_prompt: str = None) -> str: ...
```

**factory.py** — the routing:
```python
def create_provider(config: dict) -> LLMProvider:
    provider = config.get("provider", "bedrock").lower()

    if provider == "bedrock":
        return BedrockProvider(config)
    if provider in ("lmstudio", "ollama"):
        return OpenAICompatibleProvider(config)
```

**scoring.py** (usage):
```python
config = {
    "provider": os.getenv("LLM_PROVIDER", "ollama"),
    "base_url": os.getenv("LLM_BASE_URL", "http://localhost:11434/v1"),
    "model":    os.getenv("LLM_MODEL", "llama2"),
}
provider = create_provider(config)
```

Three env vars. Zero business logic changes. Full provider flexibility.

### The Three Environments, Three Strategies

| Environment | Provider | Why |
|---|---|---|
| Local dev | Ollama / LM Studio | Zero cost, offline, instant iteration |
| CI/CD | Ollama (deterministic model) | Controlled cost, no quota surprises |
| Production | AWS Bedrock | Managed, scalable, SLA-backed |

The same prompt runs in all three environments. Results differ by model capability, but the pipeline structure is identical.

### What This Protects Against

- **Quota exhaustion** — dev load never touches production quotas
- **Cost overruns** — CI doesn't burn cloud credits on every push
- **Vendor lock-in** — switching Bedrock for a competitor is a one-line env change
- **Latency constraints** — route to local for speed, cloud for quality
- **Experimental uncertainty** — test new models without touching business logic

### The Comparison That Made This Click

Before building this, I had the Bedrock client hardcoded in the freelance-automation-hub project. Every test run cost money. CI was expensive. Running locally required AWS credentials. Local iteration was slow.

After the abstraction: dev runs on a local Ollama instance. CI uses the same. Production switches to Bedrock with an env var in the deployment config. The workflow became frictionless.

### It's Not About the Code

The abstraction itself is ~100 lines. That's not the point.

The point is the *discipline*: treating model providers as infrastructure means you're thinking about environments, cost, resilience, and portability — not just "which API do I call."

Most AI content covers prompt engineering and tool selection. Fewer people talk about what happens when you need to run the same LLM-integrated system across five engineers, a CI pipeline, a staging environment, and production — with different cost envelopes and availability requirements at each layer.

That's the infrastructure problem. And it has infrastructure solutions.

---

## Diagram

![LLM Architecture Diagram](./post-01-diagram.svg)

**Caption**: Provider-Agnostic LLM Architecture — Same code, different backends, one env var.

---

## Source Projects

- **project-recon** — Freelance project discovery pipeline
- **freelance-automation-hub** — Automated freelance project application
