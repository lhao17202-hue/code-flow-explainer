---
name: code-flow-explainer
description: Use this skill whenever the user wants to deeply understand how code executes at runtime, especially requests like “walk through this step by step”, “trace the data flow”, “simulate this function”, “what are the inputs and outputs”, “how is this variable produced”, “what does this call return”, “按执行顺序讲”, “从数据流讲”, “模拟运行”, or “这个变量怎么来的”. Explain by tracing the exact runtime path from the call site through parameter binding, intermediate values, branches, helper calls, side effects, return values, and downstream usage. Prefer concrete example values and before/after states over abstract summaries.
---

# Code Flow Explainer

Use this skill to explain code by reconstructing what happens at runtime.

The goal is not to summarize what the code “does” in one paragraph. The goal is to help the user build a mental model of execution: what value enters, where it goes, how it changes, what code runs next, what state is read or mutated, what object is returned, and how that returned value is used later.

## Core approach

When explaining code, reconstruct the runtime story:

1. Start at the call site.
2. Identify the direct input and output.
3. Set a concrete runtime scenario when helpful.
4. Bind actual input values to function parameters.
5. Execute the function in source order.
6. Track intermediate variables and object state.
7. Explain branches, loops, fallbacks, and exceptions.
8. Identify side effects such as file I/O, network calls, subprocesses, environment access, database writes, logging, caching, or mutation.
9. Construct the final returned value.
10. Connect the return value to downstream code that uses it.
11. End with a concise mental model.

Prefer concrete execution tracing over high-level paraphrase.

Bad style:

> This function builds a context object with metadata.

Better style:

> The caller passes `args.cwd`. If `args.cwd = "."` and the process is running in `/repo/app`, then `Path(cwd).resolve()` changes the value from `"."` to `Path("/repo/app")`. Next, the nested helper function is defined. No helper command has run yet. Then, because `repo_root_override` is `None`, execution continues into the `else` branch...

## When to use

Use this skill when the user asks questions like:

- “Walk through this code step by step.”
- “Trace the data flow.”
- “Simulate this function running.”
- “What does this line return?”
- “What are the inputs and outputs?”
- “How is this variable produced?”
- “What happens after this function is called?”
- “Explain this from the call site.”
- “Show the before and after values.”
- “按执行顺序讲.”
- “从数据流讲.”
- “一步一步讲.”
- “模拟运行全过程.”
- “这个变量怎么来的？”
- “这个函数最后得到了什么？”

Also use it when the user is confused about runtime concepts such as constructors, class methods, instance methods, decorators, context managers, callbacks, async/await, subprocess calls, filesystem paths, config loading, environment variables, caches, session objects, dependency injection, or state mutation.

## Explanation structure

Use this structure unless the user asks for a different format.

### 1. State the target

Quote the exact line, function, method, class, or code block being explained.

Briefly say what question this code answers.

Example:

```python
result = Client.build(config.path)
```

> This line turns the CLI/config value `config.path` into a constructed `Client` object.

### 2. Identify direct input and output

Name the input values entering the call.

Example:

```python
config.path
```

Name the output variable or return value.

Example:

```python
result: Client
```

If the type is inferred rather than explicit, say so.

### 3. Set a concrete runtime scenario

When possible, choose realistic values and state them up front.

Example:

```text
config.path = "."
current process directory = /repo/app
```

Then rewrite the call with those values:

```python
Client.build(".")
```

Concrete values reduce ambiguity and make execution easier to follow.

### 4. Enter the called function

Show the function signature.

Example:

```python
@classmethod
def build(cls, path, override=None):
```

Explain parameter binding:

```python
cls = Client
path = "."
override = None
```

If relevant, briefly explain the mechanism:

- For `@classmethod`, the class is passed as `cls`.
- For instance methods, the object is passed as `self`.
- For constructors, `__init__` initializes the newly created object.
- For decorators, the decorated function may be wrapped before execution.
- For context managers, `__enter__` and `__exit__` affect control flow.
- For async functions, calling may create a coroutine until it is awaited.

### 5. Trace execution in source order

Walk line by line or block by block.

For each important line, explain:

1. what executes,
2. what input value it uses,
3. what value it produces,
4. whether it changes state,
5. whether it calls an external system,
6. what happens if it fails.

Use before/after snapshots when helpful.

Example:

```python
path = Path(path).resolve()
```

Before:

```python
path = "."
```

After:

```python
path = Path("/repo/app")
```

Meaning:

> The relative path is normalized into an absolute path.

### 6. Track intermediate variables

Whenever a variable is created or changes meaning, show its value.

Example:

```python
items = {}
```

Later:

```python
items = {
    "README.md": "first 1200 characters...",
    "config.json": "first 1200 characters...",
}
```

For collections, show representative structure rather than dumping huge content.

### 7. Explain branches and fallbacks

For conditionals, explain the path taken in the concrete scenario and mention important alternate paths.

Example:

```python
root = override if override is not None else detect_root(path)
```

Explain:

- If `override` is provided, the code uses it directly.
- If `override` is `None`, the code computes the root from `path`.
- In the current scenario, `override = None`, so the second path runs.

For `try`/`except`, explain both success and failure behavior.

### 8. Explain helper functions when they are defined and when they are called

If code defines a helper function, distinguish definition from execution.

Example:

```python
def run(command, fallback=""):
    ...
```

Say:

> At this point, the helper function is only defined. No command has run yet.

Then, when the helper is called:

```python
run(["status"], "clean")
```

Explain the actual operation:

```text
This executes the helper with args=["status"] and fallback="clean".
```

If it wraps an external command, show the effective command.

### 9. Separate metadata reads from content reads

Be precise about what the code reads.

Examples:

- Reading a directory listing is not the same as reading every file.
- Asking a version-control system for the current branch is not the same as loading every version of the project.
- Reading an environment variable is not the same as reading a config file.
- Creating a path object is not the same as opening that path.

This distinction prevents false mental models.

### 10. Identify side effects

Explicitly say whether the code has side effects.

Common side effects include:

- reading files,
- writing files,
- creating directories,
- deleting files,
- executing subprocesses,
- sending network requests,
- reading or modifying environment variables,
- reading or writing databases,
- mutating objects,
- changing global state,
- logging or tracing,
- caching.

If the code does not mutate anything externally, say that too.

Example:

> This function reads a few files and executes read-only commands, but it does not modify source files.

### 11. Construct the final returned value

At the end, show the approximate shape of the returned value.

Example:

```python
result = ClientContext(
    cwd="/repo/app",
    root="/repo",
    mode="dev",
    status="clean",
    docs={
        "README.md": "...",
    },
)
```

Then show how the caller receives it:

```python
context = ClientContext.build(config.path)
```

After the right-hand side returns:

```python
context.cwd
context.root
context.status
```

### 12. Explain downstream usage

Do not stop at “it returns X”. Explain why X matters in the surrounding code.

Look for subsequent uses of the returned value. Explain how later code depends on it.

Examples:

- A root path may determine where config is loaded from.
- A context object may define the base directory for tools.
- A client object may be used for later API calls.
- A parsed config may determine feature flags.
- A cache key may decide whether cached data can be reused.

### 13. End with a compact mental model

Finish with one or two sentences that compress the runtime story.

Example:

> In one sentence: this call converts a user-provided path into a runtime context object. The object records the resolved path, detected project state, and small pieces of metadata that later code uses as its coordinate system.

## Handling missing context

If the user only provides one line and the function/class definition is missing, inspect or ask for the relevant definition before giving a deep trace.

If enough code is visible in the conversation or selected text, proceed directly. Do not ask for more context unless the missing code would materially change the answer.

If multiple execution paths are possible and the user has not specified runtime values, choose a clear representative scenario, label it as an assumption, and also describe important alternate branches.

## Tone and teaching style

Use clear headings and numbered steps.

Prefer plain language over jargon. When a term is useful, define it briefly the first time.

Good terms to define when needed:

- call site: the place where a function is called
- parameter binding: assigning passed values to function parameters
- side effect: something the code changes or performs outside the local return value
- fallback: a default value or path used when the preferred operation fails
- working directory: the directory relative paths are resolved from
- return value: the value sent back to the caller

Use code blocks for:

- input values,
- intermediate values,
- before/after states,
- final object shapes,
- commands,
- important source snippets.

## Common mistakes to avoid

Do not only summarize intent.

Do not skip the call site.

Do not skip parameter binding.

Do not treat function definition as function execution.

Do not skip fallback and error paths.

Do not imply that metadata inspection reads all content.

Do not describe side effects vaguely. Say exactly what is read, written, executed, or mutated.

Do not stop before explaining the return value’s downstream use.

Do not overfocus on syntax when the user is asking about runtime behavior.

## Reusable response template

```markdown
这段代码/这行调用是：

```language
...
```

## 1. 输入是什么？

...

## 2. 输出是什么？

...

## 3. 假设一个具体运行场景

```text
...
```

## 4. 进入函数后，参数如何绑定？

```language
...
```

## 5. 按执行顺序发生了什么？

### Step 1: ...

Before:

```language
...
```

After:

```language
...
```

解释：...

### Step 2: ...

...

## 6. 中间变量如何变化？

...

## 7. 分支、fallback、异常路径是什么？

...

## 8. 有哪些副作用？

...

## 9. 最后返回什么？

```language
...
```

## 10. 返回值后面怎么用？

...

## 11. 一句话理解

...
```

Adapt the headings to the user’s language. If the user asks in Chinese, answer in Chinese unless they request otherwise.
