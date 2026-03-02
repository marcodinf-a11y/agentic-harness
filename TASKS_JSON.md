# Agentic Harness — Task Definition Spec (JSON)

Tasks are defined as JSON files. Each file describes a single unit of work to dispatch to an agent.

## Why JSON

Task files are **human-written, agent-read** — the developer authors them, the harness feeds them to agents. JSON is the MVP task format. YAML support is planned as a future addition — see [TASKS_YAML.md](TASKS_YAML.md) for the rationale and planned spec.

### Generation reliability

JSON has the highest structured output reliability across all LLMs. StructEval benchmarks show GPT-4o at 99.36% accuracy for JSON generation. All three providers (Anthropic, OpenAI, Google) support only JSON for constrained decoding. Source: [StructEval (arXiv)](https://arxiv.org/html/2505.20139v1).

### Training data prevalence

JSON is vastly more prevalent in LLM training corpora — it appears embedded in virtually every programming language. This training data advantage translates to more reliable parsing and understanding, especially in smaller models.

### Strict syntax

JSON has no implicit type coercion, no indentation-sensitive parsing, and one unambiguous representation for any given structure. A malformed JSON file fails loudly at parse time — there are no silent misinterpretations.

### Tooling

JSON Schema validation is a mature ecosystem. Every language has robust JSON parsers. IDE support (syntax highlighting, formatting, linting) is universal.

### Why not YAML?

YAML wins on readability, comments, and multiline strings. The ImprovingAgents benchmark found YAML outperformed JSON on comprehension accuracy for 2 of 3 models, and YAML uses ~24% fewer tokens. If the developer experience of authoring task files is the priority, see [TASKS_YAML.md](TASKS_YAML.md). The tradeoff: YAML's indentation sensitivity and implicit type coercion introduce silent failure risks that JSON avoids entirely.

### Known JSON limitations (mitigated)

- **No comments**: mitigated by using `metadata` field for annotations, or a companion README per task directory
- **Multiline strings**: mitigated by `\n` escapes — less readable but unambiguous
- **Verbosity**: ~24% more tokens than YAML — a cost tradeoff, not a correctness issue

## JSON Schema

```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "required": ["id", "name", "prompt"],
    "properties": {
        "id":                  { "type": "string", "description": "Unique task identifier" },
        "name":                { "type": "string", "description": "Human-readable task name" },
        "prompt":              { "type": "string", "description": "Prompt sent to the agent" },
        "setup_commands":      { "type": "array", "items": { "type": "string" }, "default": [] },
        "validation_commands": { "type": "array", "items": { "type": "string" }, "default": [] },
        "files":               { "type": "object", "additionalProperties": { "type": "string" }, "default": {} },
        "token_budget":        { "type": "integer", "default": 70000 },
        "timeout_seconds":     { "type": "integer", "default": 300 },
        "tags":                { "type": "array", "items": { "type": "string" }, "default": [] },
        "metadata":            { "type": "object", "default": {} }
    },
    "additionalProperties": false
}
```

### Required Fields

- `id` — must be unique across all task files
- `name` — human-readable label
- `prompt` — the instruction sent to the agent

### Optional Fields (with defaults)

- `setup_commands` — default: `[]`
- `validation_commands` — default: `[]`
- `files` — default: `{}`
- `token_budget` — default: `70000`
- `timeout_seconds` — default: `300`
- `tags` — default: `[]`
- `metadata` — default: `{}`

## Example: FizzBuzz

```json
{
    "id": "fizzbuzz-001",
    "name": "FizzBuzz implementation",
    "prompt": "Create a Python file called fizzbuzz.py that implements FizzBuzz for numbers 1-100.\nPrint \"Fizz\" for multiples of 3, \"Buzz\" for multiples of 5, \"FizzBuzz\" for both.\nInclude a main() function and if __name__ == \"__main__\" guard.",
    "setup_commands": [
        "git init",
        "python3 -m venv .venv"
    ],
    "validation_commands": [
        "python3 fizzbuzz.py | head -15",
        "python3 -c \"import fizzbuzz; fizzbuzz.main()\""
    ],
    "files": {
        "requirements.txt": ""
    },
    "token_budget": 70000,
    "timeout_seconds": 120,
    "tags": ["easy", "python", "basics"],
    "metadata": {
        "expected_files": ["fizzbuzz.py"],
        "difficulty": "easy"
    }
}
```

## Example: REST API

```json
{
    "id": "rest-api-001",
    "name": "Flask REST API with tests",
    "prompt": "Create a Flask REST API with:\n- GET /items - list all items\n- POST /items - create an item (json body: {\"name\": \"...\", \"price\": ...})\n- GET /items/<id> - get item by ID\n- DELETE /items/<id> - delete item\nUse an in-memory dict for storage. Include pytest tests in test_app.py.",
    "setup_commands": [
        "git init",
        "python3 -m venv .venv",
        ".venv/bin/pip install flask pytest"
    ],
    "validation_commands": [
        ".venv/bin/python -c 'import app'",
        ".venv/bin/pytest test_app.py -v"
    ],
    "files": {
        "requirements.txt": "flask\npytest\n"
    },
    "token_budget": 70000,
    "timeout_seconds": 300,
    "tags": ["medium", "python", "api", "testing"],
    "metadata": {
        "expected_files": ["app.py", "test_app.py"],
        "difficulty": "medium"
    }
}
```

## Corresponding Python Dataclass

```python
@dataclass(frozen=True)
class TaskDefinition:
    id: str
    name: str
    prompt: str
    setup_commands: list[str] = field(default_factory=list)
    validation_commands: list[str] = field(default_factory=list)
    files: dict[str, str] = field(default_factory=dict)
    token_budget: int = 70_000
    timeout_seconds: int = 300
    tags: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
```

The dataclass is format-agnostic. When YAML support is added, the loader will detect the file extension and parse accordingly.

## Validation Patterns

After the agent completes, the harness runs each entry in `validation_commands` inside the sandbox. Each command is executed as a shell subprocess.

### Scoring

Scoring is binary based on exit codes:

- **All commands exit 0** → score = `1.0` (pass)
- **Any command exits non-zero** → score = `0.0` (fail)

Stdout/stderr from validation commands is captured in the report for debugging.

### Common Validation Patterns

```json
{
    "validation_commands": ["test -f output.py"]
}
```

```json
{
    "validation_commands": ["python3 main.py"]
}
```

```json
{
    "validation_commands": ["pytest tests/ -v"]
}
```

```json
{
    "validation_commands": [
        "python3 -c 'import mymodule'",
        "pytest tests/ -v",
        "ruff check ."
    ]
}
```

## Token Budget Guidelines

### Default Budget: 70,000 tokens

The budget covers `input_tokens + output_tokens`. Thinking/reasoning tokens and cache tokens are excluded.

### Budget Thresholds

| Status | Range | Meaning |
|---|---|---|
| **WITHIN** | < 80% (< 56,000) | Task is progressing normally |
| **WARNING** | 80–100% (56,000–70,000) | Context rot risk elevated, consider wrapping up |
| **EXCEEDED** | > 100% (> 70,000) | Context rot likely, start fresh |

### Choosing a Budget

- **Simple tasks** (single file, clear spec): 30,000–50,000 tokens
- **Medium tasks** (multi-file, some ambiguity): 70,000 tokens (default)
- **Complex tasks** (multi-file, testing, iteration): 70,000–100,000 tokens

If a task consistently exceeds its budget, break it into smaller tasks rather than raising the budget. Context rot doesn't care about your intentions.

### Override

Per-task via JSON:

```json
{
    "token_budget": 50000
}
```

Per-run via CLI:

```bash
harness run -t tasks/my_task.json --budget 50000
```

## File Seeding

The `files` object seeds the sandbox before the agent runs. Keys are relative paths, values are file contents.

```json
{
    "files": {
        "requirements.txt": "",
        "config.json": "{\"debug\": true, \"port\": 8080}",
        "src/utils.py": "def helper():\n    pass\n"
    }
}
```

Directories are created automatically. This is useful for:

- Providing starter code the agent should build on
- Creating config files the agent needs
- Setting up the expected project structure

## Task Directory Convention

Store tasks in `tasks/`:

```
tasks/
    example_fizzbuzz.json
    rest_api.json
    refactor_auth.json
```

Run a single task or an entire directory:

```bash
harness run -t tasks/example_fizzbuzz.json -a claude
harness run -t tasks/ -a claude
```

The MVP harness accepts `.json` files. When scanning a directory, it loads all `.json` files. YAML support (`.yaml`) is planned for a future release.
