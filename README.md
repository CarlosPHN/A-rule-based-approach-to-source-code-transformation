# GreenCodeRefactor

**Rule-based, AST-driven automatic code refactoring for energy-efficient software.**

GreenCodeRefactor applies behavior-preserving source-code transformations aimed at reducing energy consumption. It includes implementations in Java (using JavaParser) and Python (using the standard-library ast module).

## 1. Available transformations

The transformations are based on [*Evaluating the Energy Efficiency of Optimization Techniques in C, Python, and Java*](https://doi.org/10.1002/spe.70096).

- **Dead Code Elimination:** `UnusedLocalVariableTransformer` and `UnusedAssignmentTransformer`.
- **Explicit Raise to Except Body:** `ExplicitRaiseToExceptBodyTransformer`.

| Transformer | Reference | What it does |
|---|---|---|
| `UnusedLocalVariablesTransformer` | Technique T9 — Dead Code Elimination; Sonar RSPEC-1481 | Detects and removes local variables that are never read. Side-effecting initializers (e.g., method calls) are preserved as standalone statements. |
| `UnusedAssignmentTransformer` | Technique T9 — Dead Code Elimination; Sonar RSPEC-1854 | Removes dead stores: assignments whose values are overwritten before being read. |
| `ExplicitRaiseToExceptBodyTransformer` | Technique T10 — Explicit raise to except body | Replaces an explicit `raise SomeException(...)` inside a `try` block, caught by a matching `except` without an alias, with the body of the `except` handler. |

### 3.1 `UnusedLocalVariablesTransformer` (RSPEC-1481)

```python
# Before
def operate():
    a = 5
    temp = a * 2     # never read
    message = "hi"   # never read
    return a

# After
def operate():
    a = 5
    return a
```

### 3.2 `UnusedAssignmentTransformer` (RSPEC-1854)

```python
# Before                          # After
def f():
    b = 2         # dead store     def f():
    b = 3                       #     b = 3
    b = 10                        #     b = 10
    return b                      #     return b
```

### 3.3 `ExplicitRaiseToExceptBodyTransformer` (T10)

```python
# Before                                                  # After
def process_data(x, data):                                def process_data(x, data):
    result = [0] * len(data)                                  result = [0] * len(data)
    for i in range(len(data)):                                for i in range(len(data)):
        try:                                                      if x == 1:
            if x == 1:                                                  result[i] = data[i] * 2
                result[i] = data[i] * 2                              else:
            else:                                                       result[i] = data[i] * 3
                raise MyException("x is not 1")                 operate()
        except MyException:                                     return result
            result[i] = data[i] * 3
    operate()
    return result
```

The transformer also removes custom exception classes and imports that become unused after
the transformation.

---

## 4. Repository layout

```
greencoderefactor/
├── greencoderefactor-java/     # Java implementation
├── greencoderefactor-python/   # Python implementation
├── llm-based/                  # LLM-based validation experiments
└── energy-results/             # Energy measurement results
```

---

## 5. Evaluation

### 5.1 Benchmark setup

The benchmark is located in `greencoderefactor-python/repo/` and contains:

- `operations.py` — unused variables and redundant assignments.
- `process_data.py` — exception handling inside a loop.
- `test_data.py` — benchmark suite used to measure execution time and energy consumption.

### 6.2 Energy measurement methodology

Energy consumption was measured using `perf stat` with RAPL counters. The original and transformed benchmarks were each executed 30 times. Results are stored in `energy-results/`, with aggregated statistics in `energy-results/plots/energy_summary.csv`.

### 6.3 Results

| Metric | Original | Transformed | Reduction |
|---|---:|---:|---:|
| Mean energy consumption (J) | 2,821.877 | 628.716 | 77.7% |
| Mean execution time (s) | 114.38 | 26.77 | 76.6% |

Both energy consumption and execution time decreased significantly after applying the transformations.

The results suggest that removing exception handling from the inner loop (T10) and eliminating dead code (T9) contributed to the improvements.

Figures are available in `energy-results/plots/`.

### 6.4 LLM-based validation

The `llm-based/` folder contains 30 independent iterations in which an LLM applied refactoring transformations to the source code. Each iteration stores the agent's output, execution statistics and transformed code.

All 30 iterations passed functional validation, although two contained an incomplete Dead Code Elimination transformation.