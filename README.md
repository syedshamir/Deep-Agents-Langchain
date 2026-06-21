# Deep Agents LangChain

A hands-on LangChain Deep Agents project that explores how to build agents for
multi-step work using planning, virtual files, context engineering, backends,
subagents, skills, and a Streamlit chatbot demo.

The project is built around the `deepagents` library and LangGraph. It starts
with simple agent creation, then moves into the parts that make deep agents
useful for real workflows: saving intermediate work, loading project context,
choosing where memory lives, delegating work to subagents, and packaging
reusable skills.

## What This Project Demonstrates

- Creating a basic deep agent with `create_deep_agent`.
- Adding custom tools, including Tavily-powered web search.
- Using model provider strings such as OpenAI, Groq, and Ollama-compatible
  models through LangChain.
- Managing context with `AGENTS.md`, `memory=`, checkpointers, and thread IDs.
- Understanding Deep Agents backends:
  `StateBackend`, `FilesystemBackend`, `StoreBackend`, and related patterns.
- Using virtual file tools such as `read_file`, `write_file`, `edit_file`,
  `ls`, `glob`, and `grep`.
- Defining and using subagents for focused research and structured outputs.
- Exploring skills for Python, AWS, LangGraph, and report writing.
- Running a Streamlit chatbot that combines the notebook concepts into one UI.

## Why Deep Agents?

Simple agents are useful when a task can be solved in a short tool-calling
loop. Deep Agents are designed for longer workflows where the model needs a
working environment.

A Deep Agent can:

- plan work with a todo list,
- write notes and intermediate outputs to files,
- read project context from files like `AGENTS.md`,
- use different storage backends for temporary or persistent memory,
- delegate isolated tasks to subagents,
- load reusable skills for domain-specific behavior.

The core idea is context engineering: give the agent the right information, in
the right place, at the right time.

## Project Structure

```text
.
+-- deepagentsdemo/
|   +-- 1-basicdeepagents.ipynb
|   +-- 2-contextengineering.ipynb
|   +-- 3-backends.ipynb
|   +-- 4-subagents.ipynb
|   +-- notes/
|   |   +-- todo.txt
|   +-- projects/
|   |   +-- AGENTS.md
|   +-- skills/
|       +-- aws/
|       +-- langgraph/
|       +-- python/
|       +-- report-writer/
+-- streamlit_app.py
+-- main.py
+-- pyproject.toml
+-- requirements.txt
+-- uv.lock
+-- README.md
```

## Notebook Guide

### `1-basicdeepagents.ipynb`

Introduces the basics:

- loading environment variables,
- creating Tavily web search as a custom tool,
- initializing LangChain chat models,
- comparing a simple agent with a deep agent,
- invoking a deep agent and inspecting generated files.

### `2-contextengineering.ipynb`

Focuses on context engineering:

- using `AGENTS.md` as durable project context,
- loading memory into the agent,
- using `MemorySaver` and thread IDs,
- seeding files into the in-state filesystem,
- experimenting with skills loaded from `/skills/`.

### `3-backends.ipynb`

Explains where Deep Agents files and memory actually live:

- `StateBackend` for thread-scoped scratch work,
- `FilesystemBackend` for real files on disk,
- `StoreBackend` for cross-thread memory,
- the mental model of a virtual filesystem backed by different storage layers.

### `4-subagents.ipynb`

Demonstrates delegation:

- defining a research subagent,
- giving a subagent its own tools and prompt,
- using subagents for context isolation,
- returning structured research results with a Pydantic response format.

## Streamlit Demo

The project includes `streamlit_app.py`, a chatbot UI that combines the main
concepts from the notebooks:

- custom model selection,
- Tavily internet search,
- `AGENTS.md` memory loading,
- skills from `deepagentsdemo/skills`,
- subagents and structured-output research,
- backend selection between state, filesystem, and store,
- thread memory through a LangGraph checkpointer,
- visibility into tool calls, todos, file operations, and subagent activity.

Run it with:

```bash
streamlit run streamlit_app.py
```

## Setup

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd Deep-Agents-Langchain
```

### 2. Create and activate a virtual environment

Using `uv`:

```bash
uv sync
```

Using `pip`:

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=your_openai_key
GROQ_API_KEY=your_groq_key
TAVILY_API_KEY=your_tavily_key
```

You only need the keys required by the model provider and tools you plan to
use. Tavily is required for the web search examples.

### 4. Run the notebooks

Start Jupyter from the project root:

```bash
jupyter lab
```

Then open the notebooks under `deepagentsdemo/` in order:

1. `1-basicdeepagents.ipynb`
2. `2-contextengineering.ipynb`
3. `3-backends.ipynb`
4. `4-subagents.ipynb`

### 5. Run the Streamlit app

```bash
streamlit run streamlit_app.py
```

## Key Concepts

### Planning

Deep Agents can maintain a todo list during execution. This helps the agent
break larger tasks into smaller steps and track progress while working.

### Virtual Files

Deep Agents expose file tools that let the agent save and retrieve intermediate
work. These paths may look like normal files, but the backend decides where the
content actually lives.

### Context Engineering

The project uses `AGENTS.md` as a context file. This keeps long-lived operating
instructions and project conventions in a versioned file instead of burying
them inside code.

### Backends

Backends define storage behavior:

- `StateBackend`: in-state, thread-scoped scratch files.
- `FilesystemBackend`: real files under a configured root directory.
- `StoreBackend`: memory stored in a LangGraph store and shared across
  threads.
- `CompositeBackend`: a routing pattern for combining multiple backends by
  path prefix.

### Subagents

Subagents let the main agent delegate focused work to another agent with its
own prompt, tools, and context. This keeps the main conversation cleaner and
helps isolate complex subtasks.

### Skills

Skills are reusable instruction packages. In this project, skills are included
for:

- AWS,
- LangGraph,
- Python,
- report writing.

Each skill contains a `SKILL.md` file plus supporting instructions and examples.

## Development Notes

- Python version: `>=3.12`
- Dependency management: `pyproject.toml`, `requirements.txt`, and `uv.lock`
- Environment secrets are excluded through `.gitignore`
- License: Apache License 2.0

## Suggested Learning Path

1. Start with the basic notebook to understand `create_deep_agent`.
2. Move to context engineering to see how memory and `AGENTS.md` work.
3. Study backends to understand where files and memory are stored.
4. Explore subagents to learn delegation and structured outputs.
5. Run the Streamlit app to see the concepts working together.

## Summary

This project is a practical walkthrough of LangChain Deep Agents. It shows how
to move from a simple tool-calling assistant toward an agent that can plan,
store work, load context, use memory, delegate tasks, and operate over longer
workflows.
