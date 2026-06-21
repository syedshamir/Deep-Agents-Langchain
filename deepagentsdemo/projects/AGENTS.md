# AGENTS.md - Deep Agents Context File

> This file is loaded into the deep agent's context through its file system tools
> whenever the agent is invoked. It gives the agent background knowledge about
> its own architecture, project conventions, and how it should operate.

---

## 1. What is a Deep Agent?

A deep agent is an agent designed for complex, multi-step, long-horizon work.
Unlike a simple tool-calling loop, a deep agent is built on LangGraph and is
meant to plan, store intermediate work, delegate subproblems, and synthesize
results over time.

The core pillars are:

1. Planning through a todo-writing tool.
2. File-system-backed context offloading.
3. Delegation to specialized subagents.
4. A detailed system prompt that teaches when and how to use the above.

---

## 2. Architecture Overview

```
                    +-----------------------------+
                    |         Deep Agent          |
                    |     create_deep_agent       |
                    |      built on LangGraph     |
                    +-------------+---------------+
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
          v                       v                       v
   +-------------+        +---------------+       +---------------+
   | Planning    |        | File System   |       | Subagents     |
   | write_todos |        | ls/read/write |       | task tool     |
   |             |        | edit/glob/grep|       | isolated ctx  |
   +-------------+        +---------------+       +---------------+
                                  |
                                  v
                         +------------------+
                         | Custom Tools     |
                         | e.g. web search  |
                         +------------------+
```

### 2.1 Planning (`write_todos`)

- The agent maintains a structured todo list with statuses such as
  `pending`, `in_progress`, and `completed`.
- Plans should be written before starting non-trivial work and updated as
  steps finish.
- Planning is not just for transparency. It helps the agent keep state across
  long tasks and reduce drift.

### 2.2 File System Tools

Deep Agents expose a virtual file system through tools such as:

- `ls`
- `read_file`
- `write_file`
- `edit_file`
- `glob`
- `grep`

Important details:

- These tools do not inherently mean "real disk". They talk to a configured
  backend, which may be LangGraph state, local files, a store, or another
  persistence layer.
- `read_file` can also support image reads where the backend allows it.
- Some backends also expose `execute` for shell command execution.

Context engineering rule:

- Bulky outputs such as search results, scraped pages, long notes, raw logs,
  and draft artifacts should be written to files and only summarized in the
  conversation.
- Re-read files when exact detail is needed again instead of keeping large
  content in the prompt.

### 2.3 Subagents (`task` tool)

- The main agent can delegate a self-contained task to a subagent.
- Each subagent gets an isolated context window, which reduces context
  pollution in the parent agent.
- Subagents are defined with a `name`, `description`, `prompt`, and optionally
  a restricted `tools` list.
- Only the subagent's final answer flows back to the parent agent.

### 2.4 System Prompt

- The default deep-agent system prompt is long and behavioral. It teaches the
  agent how to plan, offload, and delegate.
- Project-specific behavior can be added through `system_prompt` in
  `create_deep_agent(...)` and through context files like this one.

---

## 3. How This Project Builds Deep Agents

```python
from langchain.chat_models import init_chat_model
from deepagents import create_deep_agent

model = init_chat_model(...)

agent = create_deep_agent(
    model=model,
    tools=[internet_search],
    system_prompt="...",
    subagents=[...],
)

result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

Project notes:

- Environment variables such as model-provider keys and Tavily keys are loaded
  from `.env`.
- Web research is done with the Tavily client wrapped as a custom tool.
- If no backend is specified, `create_deep_agent(...)` uses `StateBackend()`
  by default.

---

## 4. Backend Mental Model

In Deep Agents, a backend is the storage and execution layer behind the agent's
filesystem tools. It is not the same thing as the LLM provider backend.

A backend decides:

- where files actually live,
- whether files persist only for one thread or across many threads,
- whether the agent can touch real disk,
- whether shell execution is available,
- what security boundary exists, if any.

This is the most important concept from the Deep Agents backends docs:

- the agent works against a virtual filesystem,
- the backend is the thing that makes that filesystem real.

---

## 5. Built-in Backends

### 5.1 `StateBackend`

`StateBackend` stores files in LangGraph state for the current thread.

Use it when:

- the agent needs a scratchpad,
- artifacts should persist across turns in one conversation,
- files should not be shared across threads.

Important behavior:

- This is the default backend.
- It is thread-scoped, not global.
- The supervisor and subagents share this state-backed filesystem, so files
  written by subagents remain available to the parent.

Best fit:

- temporary notes,
- intermediate plans,
- large tool outputs,
- draft artifacts that do not need cross-thread persistence.

### 5.2 `FilesystemBackend`

`FilesystemBackend` maps agent file operations to real files under a
`root_dir`.

Use it when:

- the agent must work with real project files,
- local development or coding workflows need actual disk persistence,
- a mounted volume or sandboxed workspace is available.

Important cautions:

- This is real disk access.
- Without `virtual_mode=True`, `root_dir` is not a strong security boundary.
- Secrets and irreversible edits are in scope if permissions are too broad.

Recommendation:

- Prefer `virtual_mode=True`.
- Do not use `FilesystemBackend` alone for most serious projects because
  internal Deep Agents artifacts can otherwise end up mixed into the real
  project directory.

### 5.3 `LocalShellBackend`

`LocalShellBackend` is effectively `FilesystemBackend` plus shell execution via
`execute`.

Use it when:

- the agent must run shell commands on the host,
- the environment is trusted and controlled,
- local developer workflows need direct command execution.

Important cautions:

- This is not sandboxed host execution.
- Even with `virtual_mode=True`, shell commands can still access the wider
  system.
- This should be treated as trusted local-dev power, not as a safe
  multi-tenant deployment option.

### 5.4 `StoreBackend`

`StoreBackend` stores files in a LangGraph `BaseStore`, which makes them
durable across threads.

Use it when:

- memory must persist across runs or threads,
- the agent needs durable notes or user memory,
- multiple sessions need access to shared stored artifacts.

Important details:

- On local setups, you can pass a store such as `InMemoryStore`.
- On LangSmith Deployment, the store is generally provisioned for you.
- Namespacing is critical so users or tenants do not accidentally share data.

### 5.5 `ContextHubBackend`

`ContextHubBackend` stores the filesystem in a LangSmith Context Hub repo.

Use it when:

- durable storage should be tied to LangSmith Hub commits,
- commit history for agent-facing files is useful,
- shared context and reusable skills should live in Hub-managed repos.

Important details:

- Reads are cached after the initial pull.
- Writes and edits create Hub commits.
- The agent repo mounts at filesystem root, while linked skill repos appear
  under `/skills/`.

### 5.6 `CompositeBackend`

`CompositeBackend` routes different path prefixes to different backends.

Use it when:

- some paths should be temporary,
- some should map to real files,
- some should persist across threads or sessions.

This is the most flexible and usually the best production-style setup.

---

## 6. Why `CompositeBackend` Matters

Deep Agents internally create and use artifacts such as:

- `/large_tool_results/`
- `/conversation_history/`

If the default backend points at real disk or durable storage, those internal
artifacts can end up mixed with project files or long-lived memory.

Best practice:

- use `StateBackend()` as the default,
- explicitly route only the paths that should persist or touch real
  infrastructure.

Example:

```python
from deepagents import CompositeBackend, FilesystemBackend, StateBackend, StoreBackend

backend = CompositeBackend(
    default=StateBackend(),
    routes={
        "/workspace/": FilesystemBackend(
            root_dir="/path/to/project",
            virtual_mode=True,
        ),
        "/memories/": StoreBackend(
            namespace=lambda rt: (rt.server_info.user.identity,),
        ),
    },
)
```

Why this is a strong default:

- scratch data stays thread-scoped,
- real project files live only under `/workspace/`,
- long-term memory lives only under `/memories/`,
- internal agent artifacts stay out of the real repo unless explicitly routed.

---

## 7. Routing Rules

`CompositeBackend` uses path-prefix routing.

Important rules:

- Longer prefixes win.
- A route like `"/memories/projects/"` overrides `"/memories/"`.
- `ls`, `glob`, and `grep` can aggregate across routed backends while keeping
  the visible path prefixes.
- Internal Deep Agents artifacts follow the default backend unless explicitly
  routed elsewhere.

Practical implication:

- The default backend should usually be the safest and least surprising one.

---

## 8. Namespace Factories

A namespace factory tells `StoreBackend` where in the store it should read and
write.

It receives a LangGraph `Runtime` and returns a tuple of strings used as the
store namespace.

This is how multi-user isolation is implemented.

Good namespace patterns:

- per user,
- per thread,
- per assistant,
- combined scopes such as `(user_id, thread_id)`.

Important guidance:

- In multi-user systems, always define a namespace explicitly.
- Do not rely on mutable runtime state for namespace selection.
- Namespaces should remain stable for the life of a run.

Version note:

- Newer `deepagents` versions use `lambda rt: ...`.
- Older code may use `lambda ctx: ...` or `BackendContext`.

---

## 9. Permissions, Policy Hooks, and Safety Layers

There are three separate control layers:

### 9.1 Backend

The backend defines what is technically possible.

Example:

- `LocalShellBackend` enables shell execution.

### 9.2 Permissions

Permissions are declarative allow or deny rules checked before the backend is
called.

Use them for simple rules such as:

- deny writes under `/policies/**`,
- allow reads only under `/workspace/**`,
- restrict destructive operations to narrow paths.

### 9.3 Policy Hooks

Policy hooks are custom logic wrappers or subclasses used for advanced
governance, such as:

- audit logging,
- rate limiting,
- content inspection,
- enterprise approval logic,
- organization-specific compliance controls.

Mental model:

- backend = capabilities,
- permissions = simple declarative restrictions,
- policy hooks = custom enforcement logic.

---

## 10. Custom Backends

If the built-in backends do not match the storage system you need, you can
implement your own backend.

Typical use cases:

- object stores,
- databases,
- remote file systems,
- custom enterprise storage layers.

To implement a custom backend, support `BackendProtocol` methods such as:

- `ls`
- `read`
- `write`
- `edit`
- `glob`
- `grep`

If shell execution is needed, implement `SandboxBackendProtocol`, which extends
the contract with `execute`.

Important implementation rule:

- return structured result objects with `error` fields instead of throwing raw
  exceptions whenever possible.

---

## 11. Protocol Expectations

When implementing or reasoning about a backend, the protocol behavior matters:

- `ls` should return deterministic file entries, ideally sorted by path.
- `read` supports pagination with `offset` and `limit`.
- `glob` returns matching paths.
- `grep` returns structured matches.
- `edit` should enforce uniqueness of `old_string` unless `replace_all=True`.
- errors should be returned in result objects rather than surfacing as crashes.

Conservative implementation note:

- where the docs appear to differ on whether `write` is create-only versus
  overwrite-capable, prefer the stricter contract unless you have a good reason
  to broaden behavior.

---

## 12. Migration Notes

Older code often passed backend factory functions into `create_deep_agent(...)`.
That pattern is deprecated in newer `deepagents` versions.

Prefer this:

```python
from deepagents import StateBackend

agent = create_deep_agent(
    model=model,
    backend=StateBackend(),
)
```

Instead of this older pattern:

```python
agent = create_deep_agent(
    model=model,
    backend=lambda rt: StateBackend(rt),
)
```

Similar migration guidance applies to:

- `StoreBackend(runtime, ...)` -> `StoreBackend(...)`
- `StateBackend(runtime)` -> `StateBackend()`

Namespace migration example:

```python
namespace=lambda rt: (rt.server_info.user.identity,)
```

Avoid deriving namespaces from mutable state.

---

## 13. Practical Backend Recommendations for This Project

If this project evolves into a serious coding or research agent, prefer a
composition strategy rather than a single all-powerful backend.

Recommended design:

1. Use `StateBackend()` as the default backend.
2. Route a narrow `/workspace/` prefix to `FilesystemBackend(...,
   virtual_mode=True)` if real file editing is required.
3. Route `/memories/` to `StoreBackend(...)` only if cross-thread persistence is
   needed.
4. Use `LocalShellBackend` only in trusted local development workflows.
5. Keep internal artifacts and temporary scratch data out of the real repo by
   leaving them on the default state backend.

---

## 14. Operating Guidelines for the Agent

When you, the deep agent, are invoked and this file is in your context:

1. Plan first for anything non-trivial.
2. Offload bulky intermediate work to files instead of bloating the prompt.
3. Choose the backend mental model before assuming a path is "real".
4. Treat `/workspace/`, `/memories/`, and other routed prefixes as intentional
   storage contracts, not just convenient folders.
5. Delegate to subagents for focused independent research or execution.
6. Prefer safer backends by default and escalate to disk or shell access only
   when the task truly requires it.
7. Keep final answers concise, synthesized, and grounded in the files or
   evidence produced during the run.

---

## 15. Key Takeaways

- Deep Agents work against a virtual filesystem.
- The backend determines what that filesystem actually is.
- `StateBackend` is safest and is the default.
- `FilesystemBackend` touches real disk.
- `LocalShellBackend` adds powerful but risky host command execution.
- `StoreBackend` and `ContextHubBackend` add durable persistence.
- `CompositeBackend` is usually the best real-world design because it cleanly
  separates scratch space, real files, and long-term memory.
