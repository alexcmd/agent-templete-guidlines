# 12 — LangSmith Observability

## 1. Auto-Tracing Setup

LangSmith traces all LangChain, LangGraph, and LangSmith SDK calls automatically when environment variables are set.

```bash
# .env
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_pt_your_key_here
LANGSMITH_PROJECT=my-agent-prod
LANGSMITH_ENDPOINT=https://api.smith.langchain.com   # default
```

```python
# src/my_agent/observability/setup.py
from __future__ import annotations

import os


def configure_tracing(
    project: str = "default",
    enabled: bool = True,
) -> None:
    """Configure LangSmith tracing. Call at application startup."""
    os.environ["LANGSMITH_TRACING"] = str(enabled).lower()
    os.environ["LANGSMITH_PROJECT"] = project
    # Verify connection
    if enabled:
        from langsmith import Client
        client = Client()
        try:
            client.read_project(project_name=project)
        except Exception:
            client.create_project(project)
        print(f"LangSmith tracing enabled → project: {project}")
```

---

## 2. @traceable Decorator Patterns

```python
# src/my_agent/observability/traceable_examples.py
from __future__ import annotations

import time
from typing import Any

from langsmith import traceable
from langsmith.run_trees import RunTree


# --- Basic ---
@traceable
def compute_answer(question: str) -> str:
    """Traced synchronously."""
    return f"Answer to: {question}"


# --- Async ---
@traceable
async def async_compute(question: str) -> str:
    import asyncio
    await asyncio.sleep(0.01)
    return f"Async answer to: {question}"


# --- Custom name, run_type, tags, metadata ---
@traceable(
    name="rag-pipeline-run",
    run_type="chain",
    tags=["rag", "production"],
    metadata={"pipeline_version": "2.1"},
)
async def rag_query(query: str, top_k: int = 5) -> dict[str, Any]:
    # retrieve, rerank, generate ...
    return {"answer": "...", "sources": []}


# --- Nested spans (parent-child) ---
@traceable(name="outer-pipeline")
async def outer(input_text: str) -> str:
    step1 = await inner_step_a(input_text)
    step2 = await inner_step_b(step1)
    return step2


@traceable(name="inner-step-a", run_type="retriever")
async def inner_step_a(text: str) -> str:
    return f"Retrieved: {text}"


@traceable(name="inner-step-b", run_type="llm")
async def inner_step_b(text: str) -> str:
    return f"Generated: {text}"


# --- Manually passing run_id to link with LangChain runs ---
@traceable(name="post-process")
def post_process(result: str, langsmith_extra: dict | None = None) -> str:
    """langsmith_extra lets you pass parent_run_id for linking."""
    return result.upper()
```

---

## 3. Manual RunTree for Complex Spans

```python
# src/my_agent/observability/run_tree_example.py
from __future__ import annotations

import time
from typing import Any

from langsmith.run_trees import RunTree


def run_with_manual_spans(inputs: dict[str, Any]) -> dict[str, Any]:
    """Manually instrument a multi-step pipeline with fine-grained spans."""

    # Root span
    root = RunTree(
        name="my-custom-pipeline",
        run_type="chain",
        inputs=inputs,
        project_name="my-agent",
        tags=["manual-instrumentation"],
    )
    root.post()

    try:
        # Child span: retrieval
        retrieval_run = root.create_child(
            name="vector-retrieval",
            run_type="retriever",
            inputs={"query": inputs.get("query", "")},
        )
        retrieval_run.post()

        docs = ["doc1", "doc2"]   # actual retrieval here
        retrieval_run.end(outputs={"documents": docs})
        retrieval_run.patch()

        # Child span: generation
        gen_run = root.create_child(
            name="llm-generation",
            run_type="llm",
            inputs={"prompt": str(docs)},
        )
        gen_run.post()

        answer = "Generated answer"
        gen_run.end(outputs={"answer": answer})
        gen_run.patch()

        outputs = {"answer": answer, "docs": docs}
        root.end(outputs=outputs)
        root.patch()
        return outputs

    except Exception as e:
        root.end(error=str(e))
        root.patch()
        raise
```

---

## 4. wrap_openai for Direct SDK Calls

```python
# src/my_agent/observability/wrap_openai_example.py
from __future__ import annotations

from openai import OpenAI, AsyncOpenAI
from langsmith.wrappers import wrap_openai


# Sync
client = wrap_openai(OpenAI())

def call_openai(prompt: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        # LangSmith traces this automatically
    )
    return response.choices[0].message.content or ""


# Async
async_client = wrap_openai(AsyncOpenAI())

async def async_call_openai(prompt: str) -> str:
    response = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content or ""
```

---

## 5. Callback Handlers

```python
# src/my_agent/observability/callbacks.py
from __future__ import annotations

from typing import Any
from uuid import UUID

from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult
from langsmith.run_helpers import LangChainTracer


# --- Custom callback handler ---
class MetricsCallbackHandler(BaseCallbackHandler):
    """Track token usage and latency."""

    def __init__(self) -> None:
        super().__init__()
        self.total_tokens = 0
        self.total_calls = 0
        self.latencies: list[float] = []
        self._start_times: dict[UUID, float] = {}

    def on_llm_start(self, serialized: dict, prompts: list[str], *, run_id: UUID, **kwargs: Any) -> None:
        import time
        self._start_times[run_id] = time.perf_counter()
        self.total_calls += 1

    def on_llm_end(self, response: LLMResult, *, run_id: UUID, **kwargs: Any) -> None:
        import time
        if run_id in self._start_times:
            latency = time.perf_counter() - self._start_times.pop(run_id)
            self.latencies.append(latency)

        # Track token usage from response metadata
        for generation in response.generations:
            for g in generation:
                if hasattr(g, "generation_info") and g.generation_info:
                    usage = g.generation_info.get("usage", {})
                    self.total_tokens += usage.get("total_tokens", 0)

    def on_llm_error(self, error: Exception, *, run_id: UUID, **kwargs: Any) -> None:
        print(f"LLM error on run {run_id}: {error}")

    @property
    def avg_latency(self) -> float:
        return sum(self.latencies) / len(self.latencies) if self.latencies else 0.0


# --- LangChainTracer (built-in LangSmith callback) ---
def get_langsmith_callbacks(run_name: str = "agent-run") -> list:
    """Get callbacks that trace to LangSmith with custom metadata."""
    return [
        LangChainTracer(
            project_name="my-agent",
            tags=["production"],
        ),
        MetricsCallbackHandler(),
    ]


# --- Pass callbacks to a chain ---
# chain.invoke(inputs, config={"callbacks": get_langsmith_callbacks()})
```

---

## 6. Dataset Creation and Management

```python
# src/my_agent/observability/datasets.py
from __future__ import annotations

from typing import Any

from langsmith import Client
from langsmith.schemas import Dataset, Example


def create_or_get_dataset(name: str, description: str = "") -> Dataset:
    client = Client()
    datasets = list(client.list_datasets(dataset_name=name))
    if datasets:
        return datasets[0]
    return client.create_dataset(name, description=description)


def add_examples(dataset_name: str, examples: list[dict[str, Any]]) -> list[Example]:
    client = Client()
    dataset = create_or_get_dataset(dataset_name)
    created = []
    for ex in examples:
        created.append(client.create_example(
            inputs=ex["inputs"],
            outputs=ex.get("outputs"),
            metadata=ex.get("metadata"),
            dataset_id=str(dataset.id),
        ))
    return created


def add_from_production_runs(
    dataset_name: str,
    project_name: str,
    min_score: float = 0.8,
    limit: int = 100,
) -> int:
    """Pull high-quality production runs into a dataset."""
    client = Client()
    dataset = create_or_get_dataset(dataset_name)

    runs = list(client.list_runs(
        project_name=project_name,
        run_type="chain",
        limit=limit,
        error=False,
    ))

    added = 0
    for run in runs:
        feedbacks = list(client.list_feedback(run_ids=[str(run.id)]))
        scores = [f.score for f in feedbacks if f.score is not None]
        if scores and sum(scores) / len(scores) >= min_score:
            client.create_example(
                inputs=run.inputs or {},
                outputs=run.outputs or {},
                dataset_id=str(dataset.id),
                source_run_id=run.id,
            )
            added += 1

    print(f"Added {added}/{len(runs)} runs to dataset '{dataset_name}'")
    return added
```

---

## 7. Experiment Runs with evaluate() / aevaluate()

```python
# src/my_agent/observability/evaluation.py
from __future__ import annotations

from typing import Any

from langsmith.evaluation import aevaluate, evaluate


def run_sync_eval(
    agent_fn: Any,
    dataset_name: str,
    experiment_name: str = "baseline",
) -> Any:
    """Synchronous evaluation run."""
    results = evaluate(
        agent_fn,
        data=dataset_name,
        evaluators=[],     # add evaluators below
        experiment_prefix=experiment_name,
        num_repetitions=1,
        metadata={"model": "gpt-4o", "version": "1.0"},
        max_concurrency=4,
    )
    return results


async def run_async_eval(
    agent_fn: Any,
    dataset_name: str,
    experiment_name: str = "baseline-async",
) -> Any:
    """Async evaluation run — faster for IO-bound agents."""
    results = await aevaluate(
        agent_fn,
        data=dataset_name,
        evaluators=[],
        experiment_prefix=experiment_name,
        max_concurrency=10,
    )
    return results
```

---

## 8. Custom Evaluators with @run_evaluator

```python
# src/my_agent/observability/evaluators.py
from __future__ import annotations

from langsmith.evaluation import run_evaluator
from langsmith.schemas import Example, Run


@run_evaluator
def code_syntax_evaluator(run: Run, example: Example) -> dict:
    """Check if generated code has valid Python syntax."""
    import ast

    output = (run.outputs or {}).get("code", "") or (run.outputs or {}).get("output", "")
    if not output:
        return {"key": "code_syntax", "score": 0.0, "comment": "No code output"}

    # Extract code blocks
    in_block = False
    blocks = []
    current: list[str] = []
    for line in output.splitlines():
        if "```python" in line:
            in_block = True
            current = []
        elif "```" in line and in_block:
            in_block = False
            blocks.append("\n".join(current))
        elif in_block:
            current.append(line)

    if not blocks:
        # Try the whole output as code
        blocks = [output]

    for block in blocks:
        try:
            ast.parse(block)
        except SyntaxError as e:
            return {"key": "code_syntax", "score": 0.0, "comment": f"SyntaxError: {e}"}

    return {"key": "code_syntax", "score": 1.0, "comment": "All code blocks are valid"}


@run_evaluator
def answer_completeness_evaluator(run: Run, example: Example) -> dict:
    """Check if the answer addresses all parts of the question."""
    output = (run.outputs or {}).get("output", "")
    expected = (example.outputs or {}).get("output", "") if example.outputs else ""

    if not output:
        return {"key": "completeness", "score": 0.0}

    # Simple heuristic: check if key terms from expected are in output
    if expected:
        expected_words = set(expected.lower().split())
        output_words = set(output.lower().split())
        overlap = len(expected_words & output_words) / len(expected_words) if expected_words else 0
        return {"key": "completeness", "score": round(min(overlap * 2, 1.0), 2)}

    # If no reference: score by length (proxy for completeness)
    score = min(len(output) / 500, 1.0)
    return {"key": "completeness", "score": round(score, 2)}
```

---

## 9. LLM-as-Judge with load_evaluator

```python
# src/my_agent/observability/llm_judge.py
from __future__ import annotations

from typing import Any

from langchain.evaluation import load_evaluator, EvaluatorType
from langchain_openai import ChatOpenAI


def build_criteria_evaluator(criteria: str) -> Any:
    """
    Build an LLM-as-judge evaluator for a specific criterion.
    criteria: "helpfulness" | "correctness" | "harmlessness" | "conciseness"
              or a custom dict
    """
    judge_llm = ChatOpenAI(model="gpt-4o", temperature=0)
    return load_evaluator(
        EvaluatorType.CRITERIA,
        llm=judge_llm,
        criteria=criteria,
    )


def build_qa_evaluator() -> Any:
    """QA evaluator: checks if output correctly answers the question."""
    judge_llm = ChatOpenAI(model="gpt-4o", temperature=0)
    return load_evaluator(
        EvaluatorType.QA,
        llm=judge_llm,
    )


def build_string_distance_evaluator() -> Any:
    """String similarity evaluator (no LLM needed)."""
    return load_evaluator(EvaluatorType.STRING_DISTANCE)


# Custom LLM-as-judge for multi-criteria scoring:
JUDGE_PROMPT = """\
Rate the following response on these criteria (each 0-10):
1. Technical accuracy
2. Code quality (if code present)
3. Clarity of explanation
4. Completeness

Question: {input}
Response: {prediction}
Reference (if available): {reference}

Return JSON: {{"accuracy": N, "quality": N, "clarity": N, "completeness": N, "reasoning": "..."}}"""


async def multi_criteria_judge(
    question: str,
    prediction: str,
    reference: str = "",
) -> dict[str, Any]:
    import json
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    prompt = JUDGE_PROMPT.format(
        input=question,
        prediction=prediction,
        reference=reference or "N/A",
    )
    result = await llm.ainvoke(prompt)
    try:
        scores = json.loads(result.content)
        avg = sum(v for k, v in scores.items() if k != "reasoning") / 4
        scores["average"] = round(avg / 10, 2)   # normalize to 0-1
        return scores
    except json.JSONDecodeError:
        return {"error": "Could not parse judge response", "raw": result.content}
```

---

## 10. Summary Evaluators

```python
# src/my_agent/observability/summary_evaluators.py
from __future__ import annotations

from typing import Any

from langsmith.evaluation import run_evaluator
from langsmith.schemas import Example, Run


def pass_rate_summary(runs: list[Run], examples: list[Example]) -> dict:
    """Calculate pass rate across all examples."""
    scores = []
    for run in runs:
        outputs = run.outputs or {}
        # Check if approved or passed
        if outputs.get("approved") is True:
            scores.append(1.0)
        elif outputs.get("approved") is False:
            scores.append(0.0)
        else:
            scores.append(0.5)

    pass_rate = sum(scores) / len(scores) if scores else 0.0
    return {
        "key": "pass_rate",
        "score": round(pass_rate, 3),
        "comment": f"{sum(1 for s in scores if s == 1.0)}/{len(scores)} passed",
    }


def avg_iteration_count_summary(runs: list[Run], examples: list[Example]) -> dict:
    """Average number of refinement iterations needed."""
    iterations = []
    for run in runs:
        outputs = run.outputs or {}
        it = outputs.get("iteration", 1)
        if isinstance(it, int):
            iterations.append(it)

    avg = sum(iterations) / len(iterations) if iterations else 0
    return {"key": "avg_iterations", "score": round(avg, 2)}


def error_rate_summary(runs: list[Run], examples: list[Example]) -> dict:
    """Fraction of runs that errored."""
    errors = sum(1 for r in runs if r.error)
    rate = errors / len(runs) if runs else 0.0
    return {"key": "error_rate", "score": round(rate, 3)}


# Pass these to evaluate():
# evaluate(
#     agent_fn,
#     data=dataset_name,
#     evaluators=[code_syntax_evaluator],
#     summary_evaluators=[pass_rate_summary, avg_iteration_count_summary, error_rate_summary],
# )
```

---

## 11. Online Evaluation in Production

```python
# src/my_agent/observability/online_eval.py
from __future__ import annotations

import asyncio
from typing import Any

from langsmith import Client
from my_agent.observability.llm_judge import multi_criteria_judge


async def evaluate_production_run(
    run_id: str,
    question: str,
    answer: str,
) -> None:
    """
    Evaluate a completed production run and store feedback in LangSmith.
    Call this after every agent completion in production.
    """
    client = Client()
    scores = await multi_criteria_judge(question, answer)

    # Store each dimension as separate feedback
    for key, score in scores.items():
        if key in ("reasoning", "error"):
            continue
        if isinstance(score, (int, float)):
            client.create_feedback(
                run_id,
                key=key,
                score=float(score),
                source_info={"evaluator": "llm-judge-gpt4o"},
            )

    # Store overall
    if "average" in scores:
        client.create_feedback(
            run_id,
            key="overall_quality",
            score=scores["average"],
            comment=scores.get("reasoning", ""),
        )
```

---

## 12. A/B Testing Experiments

```python
# src/my_agent/observability/ab_testing.py
from __future__ import annotations

import asyncio
from typing import Any

from langsmith.evaluation import aevaluate
from langsmith import Client


async def run_ab_test(
    dataset_name: str,
    variant_a: Any,
    variant_b: Any,
    evaluators: list,
    prefix_a: str = "variant-a",
    prefix_b: str = "variant-b",
) -> dict[str, Any]:
    """Run A/B test of two agent variants on the same dataset."""
    results_a, results_b = await asyncio.gather(
        aevaluate(variant_a, data=dataset_name, evaluators=evaluators,
                  experiment_prefix=prefix_a, max_concurrency=5),
        aevaluate(variant_b, data=dataset_name, evaluators=evaluators,
                  experiment_prefix=prefix_b, max_concurrency=5),
    )

    # Compare
    def summarize(results: Any) -> dict:
        scores: dict[str, list[float]] = {}
        for result in results:
            for ev in result.get("evaluation_results", {}).get("results", []):
                key = ev.get("key", "unknown")
                score = ev.get("score")
                if score is not None:
                    scores.setdefault(key, []).append(float(score))
        return {k: round(sum(v) / len(v), 3) for k, v in scores.items()}

    summary_a = summarize(results_a)
    summary_b = summarize(results_b)

    comparison = {
        "variant_a": summary_a,
        "variant_b": summary_b,
        "winner": {},
    }
    for metric in set(list(summary_a.keys()) + list(summary_b.keys())):
        a_score = summary_a.get(metric, 0)
        b_score = summary_b.get(metric, 0)
        comparison["winner"][metric] = "A" if a_score >= b_score else "B"

    return comparison
```

---

## 13. Hub Prompts: Pull, Push, Version Management

```python
# src/my_agent/observability/hub_prompts.py
from __future__ import annotations

from langchain import hub
from langsmith import Client


# --- Pull latest ---
react_prompt = hub.pull("hwchase17/react")

# --- Pull specific version ---
react_v1 = hub.pull("hwchase17/react:abc123")

# --- Push a new version ---
from langchain_core.prompts import ChatPromptTemplate

my_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("human", "{input}"),
])

# hub.push("my-org/my-agent-system", my_prompt)


# --- List versions ---
def list_hub_versions(prompt_repo: str) -> list[dict]:
    client = Client()
    commits = list(client.list_commits(prompt_identifier=prompt_repo))
    return [
        {
            "hash": c.commit_hash[:8],
            "created_at": str(c.created_at),
            "manifest_id": str(c.manifest_id),
        }
        for c in commits
    ]


# --- Prompt registry pattern ---
class PromptRegistry:
    """Cache Hub prompts at startup, allow version pinning."""

    def __init__(self, pinned_versions: dict[str, str] | None = None) -> None:
        self._cache: dict[str, Any] = {}
        self._pins = pinned_versions or {}

    def load(self, name: str, repo: str) -> None:
        version = self._pins.get(name)
        ref = f"{repo}:{version}" if version else repo
        self._cache[name] = hub.pull(ref)
        print(f"Loaded prompt '{name}' from {ref}")

    def get(self, name: str) -> Any:
        if name not in self._cache:
            raise KeyError(f"Prompt '{name}' not loaded. Call load() first.")
        return self._cache[name]


# Usage:
# registry = PromptRegistry(pinned_versions={"react": "abc123def"})
# registry.load("react", "my-org/react-agent")
# prompt = registry.get("react")
```
