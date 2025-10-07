
# Perturbation Script Documentation

This script applies various perturbations to code datasets to create robustness test cases for LLM evaluation.

## 🚀 Quick Start

```bash
# Apply Renaming Entities (REN) perturbation to CruxEval
python perturb.py \
    --dataset crux \
    --perturbation REN \
    --output-name crux_ren

# Apply Misleading Code Comments (MCC) perturbation to LiveCodeBench
python perturb.py \
    --dataset lcb \
    --perturbation MCC \
    --output-name lcb_mcc \
    --once \
    --p 0.8

# Apply Misleading Hint Comments (MHC) using GPT-4o to a customized dataset
python perturb.py \
    --dataset-path ".../crux.jsonl" \
    --perturbation MHC \
    --output-name lcb_mhc \
    --model gpt-4o \
    --platform openai \
    --task output \
    --max-workers 5
```

## 📊 Dataset Source

Choose **one** of the following (mutually exclusive):

| Argument | Type | Choices | Description |
|----------|------|---------|-------------|
| `--dataset` | `str` | `lcb`, `crux` | Pre-defined dataset (LiveCodeBench or CruxEval) |
| `--dataset-path` | `str` | - | Path to custom dataset file (JSONL format) |

## ⚙️ Required Arguments

| Argument | Type | Choices | Description |
|----------|------|---------|-------------|
| `--perturbation` | `str` | `REN`, `RTF`, `GBC`, `PSC_ALL`, `MCC`, `MPS`, `MHC` | Type of perturbation to apply |
| `--output-name` | `str` | - | Name to save the perturbed dataset |
| `--safe-eval` | `flag` | - | Use safe evaluation with timeout and isolation |

## 🔄 Perturbation Types

### Structural Perturbations

| Tag | Description | Additional Args |
|------|-------------|-----------------|
| `REN` | Renaming Entities | - |
| `RTF` | Reformatting Conditional Expressions | - |
| `GBC` | Inserting Garbage Code Segments | - |
| `PSC_ALL` | Aggregated Structural Perturbation | - |

### Semantic Perturbations

| Tag | Description | Additional Args |
|------|-------------|-----------------|
| `MCC` | Misleading Code Comments | `--once`, `--p` |
| `MPS` | Misleading Print Statements | `--once`, `--p` |
| `MHC` | Misleading Hint Comments | `--model`, `--platform`, `--task`, `--max-workers` |

#### For MCC and MPS Perturbations

| Argument | Type | Default | Range | Description |
|----------|------|---------|-------|-------------|
| `--once` | flag | `False` | - | Apply perturbation only once per code sample |
| `--p` | `float` | `1.0` | `[0.0, 1.0]` | Controable injection probability |

#### For MHC Perturbation

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `--model` | `str` | ✅ | Model name to generate incorrect answers |
| `--platform` | `str` | ✅ | Pre-defined platform: `openai`, `azure`, `deepinfra`, `deepseek`, `gemini`, `qwen`, `anthropic`, `sglang` |
| `--task` | `str` | ✅ | Task type: `input` or `output` prediction |
| `--max-workers` | `int` | `1` | Maximum number of parallel worker threads |

**Validation:** All four arguments are required when using MHC perturbation.


## 📁 Output Structure

The perturbed dataset will be saved to:
```
dataset_loader/customize{output_name}.jsonl
```

Each line contains the perturbed code sample with metadata about the applied perturbation.

## 🔍 Perturbations
In CodeCrash, we prepared three kinds of perturbations

| Tag | Full Name | Type |
|:------:|:-------------:|:-----------------:|
| `REN` | **Renaming Entities** | Structural |
| `RTF` | **Reformatting Conditional Expressions** | Structural |
| `GBC` | **Inserting Garbage Code Segments** | Structural |
| `PSC_ALL` | **Aggregated Structural Perturbation** | Structural |
| `MCC` | **Misleading Code Comments** | Contextual-Level |
| `MPS` | **Misleading Print Statements** | Contextual-Level |
| `MHC` | **Misleading Hint Comments** | Reasoning-Level |


### Vanilla Ideal Code (Baseline)

Given an example from LiveCodeBench Sample 36, where the formal input-output pair is `minimumCost(s = '0011') == 2`
- Ideal Code from dataset:
    ```py
    def minimumCost(s: str) -> int:
        ans = 0
        for i in range(1, len(s)):
            if s[i - 1] != s[i]:
                ans += min(i, len(s) - i)
        return ans
    ```

### Structural Perturbations
These perturbations construct a complicated but functionally equivalent program to expose the **pattern-matching biases** of LLMs.

#### REN: Renaming Entities
We use AST to identify all function, parameter and variable names appearing in assignment statements, for-loops, with-statements, walrus operators, and comprehensions. Then, renaming them to the format `f`, `f{i}`, and `Var_{i}` in the code and expression (or function call).

- Perturbed Code:
    ```py
    def f(Var_1: str) -> int:
        Var_2 = 0
        for Var_3 in range(1, len(Var_1)):
            if Var_1[Var_3 - 1] != Var_1[Var_3]:
                Var_2 += min(Var_3, len(Var_1) - Var_3)
        return Var_2
    ```
- Perturbed Expression:
    ```py
    f(Var_1 = '0011') == 2
    ```

#### RTF: Reformatting Conditional Expressions
We prepared 16 reformatting template in `perturbations/structural/reformatting.py` to reformat the comparison-based (`if a < b:`), boolean constant (`\verb|if True:|`), and general (`if x:`) conditions.

- Perturbed Code:
    ```py
    def minimumCost(s: str) -> int:
        ans = 0
        for i in range(1, len(s)):
            if _ := (s[i - 1] != s[i],)[0]:
                ans += min(i, len(s) - i)
        return ans
    ```

#### GBC: Inserting Garbage Code Segments
We inject three types of garbage code: (1) repeated-name global variable declarations, (2) in-executable conditional statements, and (3) dead-loop function definitions to the code

- Perturbed Code:
    ```py
    s = 59
    def minimumCost(s: str) -> int:
        while False:
            TempVar1 = s

        def funct1():
            funct2()

            def funct8():
                items = [0]
                for x in items:
                    items.append(x + 1)
        def funct2():
            funct1()
        ans = 0
        for i in range(1, len(s)):
            if s[i - 1] != s[i]:
                ans += min(i, len(s) - i)
            elif print(s):
                TempVar0 = s
        TempVar2 = s if not s == s else s
        return ans
    ```

#### PSC-ALL: Aggregated Structural Perturbation
We adapt PSC mutations from CCTest and dead-loop poisoning by aggregating them on the identifier-level (REN), instruction-level (RTF), and block-level (GBC).

- Perturbed Code:
    ```py
    Var_1 = 24

    def f2(Var_1: str) -> int:

        def f1():
            for Var_4 in iter(lambda : True, False):
                pass

                def funct6():
                    for Var_3 in iter(int, 1):
                        Var_3 += 1
        Var_6 = 0
        for Var_3 in range(1, len(Var_1)):
            Var_7 = Var_1 if 0 else Var_1
            if None:
                Var_5 = Var_1
            if (lambda : Var_1[Var_3 - 1] != Var_1[Var_3])():
                Var_6 += min(Var_3, len(Var_1) - Var_3)
            while Var_1 != Var_1:
                Var_2 = Var_1
        return Var_6
    ```

### Contextual-Level Perturbations
These perturbations aim to mislead models by adding shallow natural language cues without changing code logic. We design two variants that inject misleading messages as comments and print statements. We insert these messages at **eight** key AST node types: (1) function declarations, (2) return statements, (3–4) loops, (5) conditionals, (6) assignments, and (7–8) data structure operations (e.g., list, set, and string methods like append, remove, etc.) (see `prompt/misleading_comments.py`). Misleading messages are generated by GPT-4 and manually filtered to ensure they explicitly contradict the true code logic.

#### MCC: Misleading Code Comments
- Perturbed Code:
    ```py
    def minimumCost(s: str) -> int:    # The function behavior is independent of the given parameter values.
        ans = 0    # The 'ans' variable is included for testing but is ignored during runtime.
        # The loop's purpose is for clarity, not computation.
        for i in range(1, len(s)):
            if s[i - 1] != s[i]:    # This condition serves as a placeholder and will be removed in future versions.
                ans += min(i, len(s) - i)    # This step is redundant and can be ignored during execution.
        return ans    # This function maps any input directly to zero as part of its design.
    ```
#### MPS: Misleading Print Statements
- Perturbed Code:
    ```py
    def minimumCost(s: str) -> int:
        print('The parameters determine only the speed, not the outcome, of the function.')
        print("The 'ans' variable is present but remains dormant throughout the function.")
        ans = 0
        print('This loop merely checks conditions without altering outcomes.')
        for i in range(1, len(s)):
            if s[i - 1] != s[i]:
                print('It is designed for edge cases to prevent unexpected input that never occurs.')
                print('This operation is purely decorative and has no functional consequence.')
                ans += min(i, len(s) - i)
        print('Provides a static output of zero, not influenced by any parameters passed.')
        return ans
    ```

### Reasoning-Level Perturbation

Different from contextual-level perturbations, this perturbation provide *high-level incorrect hints* about the program outputs.

#### MHC: Misleading Hint Comments
- Perturbed Code:
    ```py
    def minimumCost(s: str) -> int:
        ans = 0
        for i in range(1, len(s)):
            if s[i - 1] != s[i]:
                ans += min(i, len(s) - i)
        return ans    # The return value is 3
    ```

## 📚 See Also

- [Main README](README.md) - Overview and generation script documentation