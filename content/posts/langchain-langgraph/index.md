+++
title = 'Chains and Graphs: Building LLM Apps with LangChain and LangGraph'
date = '2026-07-28T00:42:00+05:30'
draft = false
description = 'A detailed, code-first guide to LangChain LCEL and LangGraph StateGraph - when to use each, how tools and state work, and a complete ReAct agent implementation in Python.'
tags = ['AI', 'LangChain', 'LangGraph', 'Agents', 'Python', 'LCEL', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'LangChain gives you components and linear chains. LangGraph adds stateful loops for agents. Here is how both work, with copy-paste Python implementations.'
+++

<img src="/images/posts/langchain-langgraph/cover.svg" alt="LangChain versus LangGraph" width="960" height="400" />

*LangChain is the parts catalog. LangGraph is the wiring diagram that can loop.*

If you have read the [agent loop](/posts/agent-loop/) and [harness](/posts/agent-harness/) posts, you already know the shape: reason, call tools, observe, repeat. **LangChain** and **LangGraph** are popular Python frameworks for building that shape without reinventing prompts, tool schemas, and orchestration every time.

This post is code-first. You will build:

1. A linear **LCEL** chain (LangChain)  
2. Tools with `@tool`  
3. A full **LangGraph** ReAct agent (manual `StateGraph`)  
4. The same agent with `create_react_agent` (shortcut)  
5. Optional checkpointing for multi-turn memory  

APIs below match the LangGraph **1.x** style (`StateGraph`, `START`, `END`, `ToolNode`). Package names shift often - pin versions in real projects.

---

## LangChain vs LangGraph (clear split)

| | **LangChain** | **LangGraph** |
|--|---------------|---------------|
| Job | Models, prompts, tools, retrievers, parsers | Orchestrate **stateful** multi-step workflows |
| Shape | Mostly **linear** (LCEL: `a \| b \| c`) | **Graph** with branches and **cycles** |
| State | Implicit in the chain call | Explicit `TypedDict` / messages state |
| Best for | RAG Q&A, classify, summarize, ETL of prompts | Tool-using agents, retries, human-in-the-loop |
| Relationship | Provides building blocks | Nodes often *are* LangChain runnables |

Rule of thumb:

- One pass, no tools loop -> **LangChain LCEL**  
- Model must call tools zero or more times -> **LangGraph**  

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

pip install \
  langgraph \
  langgraph-prebuilt \
  langchain \
  langchain-core \
  langchain-openai \
  langchain-ollama
```

Use **OpenAI**:

```bash
export OPENAI_API_KEY="sk-..."
```

Or **local Ollama** (see [local models](/posts/local-models/)):

```bash
ollama pull qwen2.5-coder:7b
```

In code, swap the LLM:

```python
# Cloud
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Local (OpenAI-compatible API via Ollama)
from langchain_ollama import ChatOllama
llm = ChatOllama(model="qwen2.5-coder:7b", temperature=0)
```

Examples below use `ChatOpenAI`. Replace with `ChatOllama` if you stay local.

---

## Part 1 - LangChain LCEL (linear chain)

<img src="/images/posts/langchain-langgraph/lcel.svg" alt="Prompt to model to parser LCEL pipeline" width="960" height="280" />

**LCEL** (LangChain Expression Language) composes runnables with `|`.

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "You explain software topics in plain English. Be concise."),
        ("human", "Explain {topic} in 3 bullet points."),
    ]
)

chain = prompt | llm | StrOutputParser()

print(chain.invoke({"topic": "vector databases"}))
```

What happens:

1. Prompt fills `{topic}`  
2. Model generates a message  
3. Parser returns a plain string  

Add streaming:

```python
for chunk in chain.stream({"topic": "LangGraph"}):
    print(chunk, end="", flush=True)
```

### Mini RAG-style chain (retriever stub)

```python
from langchain_core.runnables import RunnablePassthrough

def fake_retrieve(query: str) -> str:
    # Replace with a real vector store retriever
    docs = {
        "refund": "Customers may return damaged items within 30 days.",
        "shipping": "Standard shipping takes 3-5 business days.",
    }
    key = "refund" if "refund" in query.lower() or "return" in query.lower() else "shipping"
    return docs[key]

rag_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "Answer using ONLY this context:\n{context}"),
        ("human", "{question}"),
    ]
)

rag_chain = (
    {
        "context": lambda x: fake_retrieve(x["question"]),
        "question": RunnablePassthrough(),
    }
    | rag_prompt
    | llm
    | StrOutputParser()
)

print(rag_chain.invoke({"question": "How long do I have to return a damaged item?"}))
```

This is still **one forward pass**. If the model needed a calculator mid-answer, LCEL alone will not loop for you. That is where LangGraph starts.

---

## Part 2 - Tools (shared by both)

Tools are normal Python functions with schemas the model can call.

```python
from langchain_core.tools import tool

@tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers and return the product."""
    return a * b

@tool
def word_count(text: str) -> int:
    """Count whitespace-separated words in text."""
    return len(text.split())

tools = [multiply, word_count]
```

Bind tools to the model so it can emit structured tool calls:

```python
llm_with_tools = llm.bind_tools(tools)
```

Fact: the **docstring** and type hints become the tool description/schema. Vague docs produce vague tool use.

---

## Part 3 - LangGraph mental model

<img src="/images/posts/langchain-langgraph/state.svg" alt="Shared AgentState updated by nodes" width="960" height="360" />

Three pieces:

1. **State** - shared data (usually a message list)  
2. **Nodes** - functions `(state) -> partial state update`  
3. **Edges** - always go next, or **conditional** route based on state  

<img src="/images/posts/langchain-langgraph/graph-loop.svg" alt="LangGraph ReAct agent tools loop" width="960" height="480" />

This is the same [agent loop](/posts/agent-loop/) you already know, expressed as a graph.

---

## Part 4 - Full ReAct agent with `StateGraph`

Complete, runnable example:

```python
from typing import Annotated, Literal

from langchain_core.messages import AIMessage, HumanMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from typing_extensions import TypedDict


# ---------- tools ----------
@tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers and return the product."""
    return a * b


@tool
def add(a: float, b: float) -> float:
    """Add two numbers and return the sum."""
    return a + b


tools = [multiply, add]
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
llm_with_tools = llm.bind_tools(tools)


# ---------- state ----------
class AgentState(TypedDict):
    # add_messages appends instead of overwriting
    messages: Annotated[list, add_messages]


# ---------- nodes ----------
def agent_node(state: AgentState) -> dict:
    """Call the LLM with the full message history."""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}


tool_node = ToolNode(tools)


def should_continue(state: AgentState) -> Literal["tools", "end"]:
    """If the last AI message has tool calls, run tools; else finish."""
    last = state["messages"][-1]
    if isinstance(last, AIMessage) and last.tool_calls:
        return "tools"
    return "end"


# ---------- graph ----------
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges(
    "agent",
    should_continue,
    {"tools": "tools", "end": END},
)
graph.add_edge("tools", "agent")  # loop back after tools

app = graph.compile()


# ---------- run ----------
result = app.invoke(
    {
        "messages": [
            HumanMessage(
                content="What is (3 * 4) + 10? Use the tools. Show the final number."
            )
        ]
    }
)

for message in result["messages"]:
    print(type(message).__name__, ":", getattr(message, "content", message))
```

### What the graph does

1. `START -> agent` - model sees the user question + tool schemas  
2. Model emits tool calls (for example `multiply(3,4)` then later `add`)  
3. `should_continue` routes to `tools`  
4. `ToolNode` executes calls and appends `ToolMessage`s  
5. Edge back to `agent`  
6. Model answers with the final number -> route to `END`  

That cycle is exactly why agents need a graph, not a single LCEL pipe.

### Inspect the compiled graph (optional)

```python
# Requires graphviz extras in some environments
print(app.get_graph().draw_mermaid())
```

---

## Part 5 - Shortcut: `create_react_agent`

For a standard tool loop, LangGraph ships a prebuilt helper:

```python
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import HumanMessage
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool


@tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b


llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
agent = create_react_agent(llm, [multiply])

out = agent.invoke(
    {"messages": [HumanMessage(content="What is 19 * 7? Use the tool.")]}
)
print(out["messages"][-1].content)
```

Use this when the default ReAct shape is enough. Drop to manual `StateGraph` when you need custom branches (reviewer node, human approval, parallel workers).

---

## Part 6 - Checkpointing (multi-turn memory)

Without a checkpointer, each `invoke` is a fresh graph run unless you pass the full history yourself.

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app_with_memory = graph.compile(checkpointer=memory)

cfg = {"configurable": {"thread_id": "demo-thread-1"}}

app_with_memory.invoke(
    {"messages": [HumanMessage(content="My favorite language is Rust.")]},
    config=cfg,
)

follow = app_with_memory.invoke(
    {"messages": [HumanMessage(content="What language did I just mention?")]},
    config=cfg,
)
print(follow["messages"][-1].content)
```

Same `thread_id` resumes the same state trail. For production, swap `MemorySaver` for a durable checkpointer (SQLite/Postgres adapters in the LangGraph ecosystem).

---

## Part 7 - Human-in-the-loop sketch

Pattern: interrupt before risky tools.

```python
from langgraph.types import interrupt  # available in recent LangGraph versions


def sensitive_tool_node(state: AgentState) -> dict:
    last = state["messages"][-1]
    # Pause for approval in apps that support resume
    decision = interrupt({"tool_calls": getattr(last, "tool_calls", [])})
    if decision.get("approve") is not True:
        return {"messages": [AIMessage(content="Action cancelled by user.")]}
    return tool_node.invoke(state)
```

Exact interrupt/resume APIs evolve - treat this as the *shape*: pause the [harness](/posts/agent-harness/), wait for a human, then continue.

---

## Part 8 - When to choose what

| Need | Choose |
|------|--------|
| Prompt -> model -> parse | LCEL chain |
| RAG one-shot Q&A | LCEL + retriever |
| Calculator / search tools in a loop | LangGraph ReAct |
| Multi-agent handoff | LangGraph custom nodes |
| Persist conversations across requests | LangGraph + checkpointer |
| Only wrapping OpenAI tool calls once | Maybe raw SDK; framework optional |

LangChain without LangGraph is still valuable. LangGraph without LangChain tools/models is uncommon - they are designed to stack.

---

## Common pitfalls

1. **Forgetting `add_messages`** - state overwrites instead of appending; history vanishes.  
2. **Missing edge `tools -> agent`** - tools run once, then the graph dies.  
3. **Vague tool docstrings** - model never calls or calls wrongly.  
4. **High temperature for tool agents** - start at `0` for demos.  
5. **Huge message history** - hit the [context window](/posts/context-windows/); summarize or trim.  
6. **Assuming LCEL can loop** - it can retry via custom code, but cycles are LangGraph's job.  

---

## How this maps to the rest of this blog

| Concept on this blog | In LangChain / LangGraph |
|----------------------|--------------------------|
| [Agent loop](/posts/agent-loop/) | `agent` node + `tools` node + conditional edge |
| [Harness](/posts/agent-harness/) | Graph runtime, tools, checkpoints, interrupts |
| [Skills](/posts/agent-skills/) | System prompts / tool choice policies you encode |
| [MCP](/posts/mcp/) | External tool servers; wrap MCP calls as `@tool`s |
| [Vector DB](/posts/vector-databases/) | Retriever node or LCEL RAG chain before the agent |

---

## Closing

**LangChain** gives you the pieces and a clean way to pipe them (`prompt | model | parser`).

**LangGraph** gives you an explicit **state machine** so an agent can call tools, observe results, and loop until done - with optional memory and human gates.

If you only remember the implementation pattern:

```text
StateGraph(state)
  .add_node("agent", call_llm)
  .add_node("tools", ToolNode(tools))
  .add_conditional_edges("agent", route)
  .add_edge("tools", "agent")
  .compile()
```

**Next step:** run Part 4 locally, then replace `multiply`/`add` with one real tool (HTTP GET, SQL read, or a [vector search](/posts/vector-databases/) function). That single swap turns the tutorial into your first domain agent.
