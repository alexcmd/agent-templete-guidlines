# Testing Agent Systems — Python / LangChain / LangGraph

Comprehensive testing for LLM agent systems requires three distinct layers: unit tests for deterministic components, integration tests against real APIs, and evaluation harnesses for model behavior. This guide covers all three, with production patterns for pytest + LangGraph.

---

## §12.1 Testing Strategy

```
Unit Tests (fast, no API calls)
  ├─ Tool schema validation
  ├─ Tool handler logic (with mocked output)
  ├─ State machine transitions
  ├─ Permission rule evaluation
  └─ Token estimation

Integration Tests (slow, real API)
  ├─ Single-turn tool calls end-to-end
  ├─ Multi-turn conversation continuity
  ├─ Compaction behavior
  └─ Memory extraction triggers

Evaluation Harnesses (judgment, expensive)
  ├─ Task success rate on benchmark suite
  ├─ Output quality scoring
  ├─ Regression detection across model versions
  └─ Cost/latency profiling
```

**Key rule**: Never mock the LLM in integration tests. Mocked LLM responses mask real prompt/schema bugs that only surface with actual API calls. Use `pytest.mark.integration` to separate and gate them in CI.

---

## §12.2 Project Structure

```
my_agent/
├── src/
│   └── my_agent/
│       ├── tools.py
│       ├── graph.py
│       ├── state.py
│       └── memory.py
└── tests/
    ├── conftest.py
    ├── unit/
    │   ├── test_tools.py
    │   ├── test_state.py
    │   └── test_tokens.py
    ├── integration/
    │   ├── test_agent_loop.py
    │   ├── test_tools_e2e.py
    │   └── test_memory.py
    └── evals/
        ├── benchmark.py
        ├── fixtures/
        │   └── tasks.json
        └── scorers.py
```

---

## §12.3 Unit Tests

### Tool Schema Validation

```python
# tests/unit/test_tools.py
import pytest
from pydantic import ValidationError
from my_agent.tools import BashTool, FileReadTool

class TestBashToolSchema:
    def test_valid_command(self):
        result = BashTool.input_schema(command="echo hello", timeout=30)
        assert result.command == "echo hello"

    def test_missing_command_raises(self):
        with pytest.raises(ValidationError):
            BashTool.input_schema()  # command is required

    def test_timeout_default(self):
        result = BashTool.input_schema(command="ls")
        assert result.timeout == 120  # default

    def test_dangerous_command_detected(self):
        tool = BashTool()
        assert tool.is_dangerous_command("rm -rf /") is True
        assert tool.is_dangerous_command("echo hello") is False
```

### Tool Handler Logic (Mocked Execution)

```python
# tests/unit/test_tools.py
from unittest.mock import AsyncMock, patch, MagicMock
from my_agent.tools import BashTool

class TestBashToolHandler:
    @pytest.fixture
    def tool(self):
        return BashTool()

    async def test_successful_execution(self, tool):
        with patch("my_agent.tools.asyncio.create_subprocess_exec") as mock_proc:
            proc = AsyncMock()
            proc.communicate.return_value = (b"hello\n", b"")
            proc.returncode = 0
            mock_proc.return_value = proc

            result = await tool.run(command="echo hello")
            assert result.output == "hello\n"
            assert result.exit_code == 0

    async def test_nonzero_exit_returns_error(self, tool):
        with patch("my_agent.tools.asyncio.create_subprocess_exec") as mock_proc:
            proc = AsyncMock()
            proc.communicate.return_value = (b"", b"command not found")
            proc.returncode = 127
            mock_proc.return_value = proc

            result = await tool.run(command="nosuchcmd")
            assert result.is_error is True
            assert "command not found" in result.output

    async def test_timeout_raises(self, tool):
        with patch("my_agent.tools.asyncio.create_subprocess_exec") as mock_proc:
            proc = AsyncMock()
            proc.communicate.side_effect = asyncio.TimeoutError()
            mock_proc.return_value = proc

            with pytest.raises(ToolTimeoutError):
                await tool.run(command="sleep 999", timeout=1)
```

### State Machine Transitions

```python
# tests/unit/test_state.py
from langgraph.graph import StateGraph, END
from my_agent.state import AgentState
from my_agent.graph import should_continue

class TestGraphTransitions:
    def test_no_tool_calls_ends(self):
        state = AgentState(
            messages=[{"role": "assistant", "content": "Done."}]
        )
        result = should_continue(state)
        assert result == END

    def test_tool_calls_continue(self):
        state = AgentState(
            messages=[{
                "role": "assistant",
                "content": [{"type": "tool_use", "name": "bash", "id": "t1", "input": {"command": "ls"}}]
            }]
        )
        result = should_continue(state)
        assert result == "tool_node"

    def test_max_iterations_ends(self):
        state = AgentState(
            messages=[],
            iteration_count=50,
        )
        result = should_continue(state)
        assert result == END
```

### Token Estimation

```python
# tests/unit/test_tokens.py
from my_agent.tokens import rough_token_estimate, token_count_with_estimation

class TestTokenEstimation:
    def test_empty_message(self):
        assert rough_token_estimate({"role": "user", "content": ""}) == 0

    def test_text_content(self):
        msg = {"role": "user", "content": "hello world"}
        # 11 chars / 4 chars_per_token * 4/3 padding = ~4 tokens
        assert 3 <= rough_token_estimate(msg) <= 6

    def test_tool_result_content(self):
        msg = {
            "role": "user",
            "content": [{"type": "tool_result", "content": "a" * 4000, "tool_use_id": "t1"}]
        }
        estimate = rough_token_estimate(msg)
        assert 1000 <= estimate <= 1500  # ~1000 tokens + 4/3 padding

    def test_with_api_usage_prefers_api(self):
        messages = [
            {"role": "user", "content": "hello"},
            {
                "role": "assistant",
                "content": "world",
                "usage": {
                    "input_tokens": 5000,
                    "output_tokens": 100,
                    "cache_creation_input_tokens": 0,
                    "cache_read_input_tokens": 0,
                }
            },
        ]
        count = token_count_with_estimation(messages)
        assert count == 5100  # exact from API
```

---

## §12.4 Integration Tests

```python
# tests/conftest.py
import pytest
import os

def pytest_configure(config):
    config.addinivalue_line(
        "markers", "integration: marks tests that call the real Anthropic API"
    )

@pytest.fixture(scope="session")
def api_key():
    key = os.environ.get("ANTHROPIC_API_KEY")
    if not key:
        pytest.skip("ANTHROPIC_API_KEY not set")
    return key

@pytest.fixture
def agent(api_key):
    from my_agent import create_agent
    return create_agent(model="claude-haiku-4-5-20251001")  # cheapest model for tests

@pytest.fixture
def thread_id():
    import uuid
    return str(uuid.uuid4())
```

```python
# tests/integration/test_agent_loop.py
import pytest

@pytest.mark.integration
async def test_single_turn_no_tools(agent, thread_id):
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "What is 2+2?"}]},
        config={"configurable": {"thread_id": thread_id}},
    )
    last = result["messages"][-1]
    assert last["role"] == "assistant"
    assert "4" in last["content"]

@pytest.mark.integration
async def test_tool_call_executed(agent, thread_id):
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Run: echo 'test-marker-12345'"}]},
        config={"configurable": {"thread_id": thread_id}},
    )
    messages = result["messages"]
    tool_results = [m for m in messages if m.get("role") == "tool"]
    assert any("test-marker-12345" in str(m.get("content", "")) for m in tool_results)

@pytest.mark.integration
async def test_multi_turn_context_preserved(agent, thread_id):
    config = {"configurable": {"thread_id": thread_id}}

    await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Remember: my name is TestUser42."}]},
        config=config,
    )
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "What is my name?"}]},
        config=config,
    )
    last = result["messages"][-1]
    assert "TestUser42" in last["content"]

@pytest.mark.integration
async def test_parallel_tools_both_execute(agent, thread_id):
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Run both: `echo A` and `echo B` simultaneously"}]},
        config={"configurable": {"thread_id": thread_id}},
    )
    messages = result["messages"]
    tool_outputs = " ".join(
        str(m.get("content", "")) for m in messages if m.get("role") == "tool"
    )
    assert "A" in tool_outputs
    assert "B" in tool_outputs
```

### Error Recovery Tests

```python
@pytest.mark.integration
async def test_tool_error_allows_recovery(agent, thread_id):
    """Model should try alternative when tool fails."""
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "Run: cat /nonexistent_file_xyz_abc.txt"}]},
        config={"configurable": {"thread_id": thread_id}},
    )
    last = result["messages"][-1]
    # Model should acknowledge the error, not crash
    assert last["role"] == "assistant"
    assert any(word in last["content"].lower() for word in ["error", "not found", "doesn't exist"])
```

---

## §12.5 Evaluation Harnesses

### Task Fixture Format

```json
// tests/evals/fixtures/tasks.json
[
  {
    "id": "bash-simple-01",
    "category": "bash",
    "input": "Count the number of Python files in the current directory",
    "expected_tool": "bash",
    "expected_output_contains": ["files", "python", ".py"],
    "scorer": "contains_all"
  },
  {
    "id": "file-read-01",
    "category": "file_ops",
    "input": "Read the README.md file and summarize it in one sentence",
    "expected_tool": "read_file",
    "scorer": "llm_judge"
  }
]
```

### Scorer Implementations

```python
# tests/evals/scorers.py
from dataclasses import dataclass
from typing import Callable
import anthropic

@dataclass
class EvalResult:
    task_id: str
    passed: bool
    score: float          # 0.0 to 1.0
    reason: str
    cost_usd: float
    latency_ms: float

def scorer_contains_all(task: dict, output: str) -> tuple[bool, float, str]:
    expected = task.get("expected_output_contains", [])
    matches = [e.lower() in output.lower() for e in expected]
    score = sum(matches) / len(matches) if matches else 1.0
    return score == 1.0, score, f"{sum(matches)}/{len(matches)} keywords found"

def scorer_tool_used(task: dict, messages: list) -> tuple[bool, float, str]:
    expected_tool = task.get("expected_tool")
    tool_calls = [
        b.get("name") for m in messages
        if m.get("role") == "assistant"
        for b in (m.get("content") if isinstance(m.get("content"), list) else [])
        if b.get("type") == "tool_use"
    ]
    used = expected_tool in tool_calls
    return used, 1.0 if used else 0.0, f"Expected {expected_tool}, saw {tool_calls}"

async def scorer_llm_judge(task: dict, output: str, client: anthropic.AsyncAnthropic) -> tuple[bool, float, str]:
    """Use Claude as a judge for open-ended outputs."""
    prompt = f"""
Task: {task['input']}
Response: {output}

Rate this response on a scale of 1-5 for correctness and completeness.
Respond with JSON: {{"score": N, "reason": "..."}}
"""
    response = await client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=200,
        messages=[{"role": "user", "content": prompt}],
    )
    import json
    data = json.loads(response.content[0].text)
    normalized = data["score"] / 5.0
    return normalized >= 0.6, normalized, data["reason"]
```

### Benchmark Runner

```python
# tests/evals/benchmark.py
import asyncio
import json
import time
from pathlib import Path
import pytest

@pytest.mark.eval
async def test_benchmark_suite(agent, tmp_path):
    tasks = json.loads(Path("tests/evals/fixtures/tasks.json").read_text())
    results = []
    client = anthropic.AsyncAnthropic()

    for task in tasks:
        start = time.monotonic()
        config = {"configurable": {"thread_id": f"eval-{task['id']}"}}

        try:
            result = await agent.ainvoke(
                {"messages": [{"role": "user", "content": task["input"]}]},
                config=config,
            )
            messages = result["messages"]
            last_content = messages[-1].get("content", "")
            latency_ms = (time.monotonic() - start) * 1000

            scorer = task.get("scorer", "contains_all")
            if scorer == "contains_all":
                passed, score, reason = scorer_contains_all(task, last_content)
            elif scorer == "llm_judge":
                passed, score, reason = await scorer_llm_judge(task, last_content, client)
            else:
                passed, score, reason = scorer_tool_used(task, messages)

            results.append(EvalResult(
                task_id=task["id"], passed=passed, score=score,
                reason=reason, cost_usd=0.0, latency_ms=latency_ms,
            ))
        except Exception as e:
            results.append(EvalResult(
                task_id=task["id"], passed=False, score=0.0,
                reason=str(e), cost_usd=0.0, latency_ms=0.0,
            ))

    # Write results
    output = tmp_path / "eval_results.json"
    output.write_text(json.dumps([vars(r) for r in results], indent=2))

    pass_rate = sum(1 for r in results if r.passed) / len(results)
    avg_score = sum(r.score for r in results) / len(results)

    print(f"\nPass rate: {pass_rate:.1%} | Avg score: {avg_score:.2f}")
    print(f"Results: {output}")

    # Fail if below baseline
    assert pass_rate >= 0.80, f"Pass rate {pass_rate:.1%} below 80% baseline"
```

---

## §12.6 LangSmith Integration for Tests

```python
# conftest.py — add LangSmith tracing for all integration tests
import os
from langsmith import Client

@pytest.fixture(autouse=True, scope="session")
def langsmith_project():
    project = os.environ.get("LANGSMITH_PROJECT", "agent-tests")
    os.environ.setdefault("LANGCHAIN_PROJECT", project)
    os.environ.setdefault("LANGCHAIN_TRACING_V2", "true")
    return project
```

Traces from test runs appear in LangSmith automatically. Filter by project to compare pass/fail across model versions.

---

## §12.7 CI Configuration

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: pytest tests/unit/ -v

  integration:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: pytest tests/integration/ -v -m integration
```

---

## §12.8 Pytest Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
markers = [
    "integration: calls real Anthropic API",
    "eval: runs full benchmark suite",
]
testpaths = ["tests"]
```

---

*[← LangSmith Observability](./12-langsmith-observability.md) | [Index →](./00-index.md)*
