# LangGraph Cheatsheet

## Mental Model

LangGraph is a framework for building **stateful, cyclic agent workflows** on top of LangChain. Where LCEL chains are linear (`A → B → C`), LangGraph models workflows as a **directed graph** where nodes are functions and edges are transitions — including conditional edges (if/else routing) and cycles (loops for reflection, retry, tool use). The key abstraction is a **State** object that flows through the graph and is updated by each node. Think of it as a state machine where each state transition can call an LLM, a tool, or any Python function.

**When to use LangGraph over LCEL chains:**
- The agent needs to loop (retry, reflect, self-correct)
- Control flow depends on LLM output (route to different nodes based on response)
- You need human-in-the-loop checkpoints
- Multiple agents need to coordinate with shared state
- You want persistence across sessions (resumable workflows)

---

## Install & Minimal Setup

```bash
pip install langgraph langchain-core langchain-openai
pip install langgraph-checkpoint-sqlite  # for persistence (optional)
```

---

## Core Concepts

### 1. State

The shared data structure that flows through every node. Defined as a `TypedDict`. Each node receives the current state and returns a partial update.

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

# Simple state
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]  # add_messages = append, not overwrite
    user_id: str
    context: str | None
    iterations: int
```

`Annotated[list, add_messages]` means: when a node returns `{"messages": [new_msg]}`, LangGraph appends to the list instead of replacing it. This is the standard pattern for chat history.

### 2. Nodes

Plain Python functions (sync or async) that receive the full state and return a dict with the fields to update.

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def call_llm(state: AgentState) -> dict:
    messages = state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}   # appended via add_messages

def increment_counter(state: AgentState) -> dict:
    return {"iterations": state["iterations"] + 1}

async def async_node(state: AgentState) -> dict:
    result = await some_async_operation()
    return {"context": result}
```

### 3. Building a Basic Graph

```python
from langgraph.graph import StateGraph, START, END

# 1. Create the graph with the state schema
builder = StateGraph(AgentState)

# 2. Add nodes
builder.add_node("llm", call_llm)
builder.add_node("tools", execute_tools)

# 3. Add edges
builder.add_edge(START, "llm")    # entry point
builder.add_edge("llm", END)      # exit

# 4. Compile
graph = builder.compile()

# 5. Invoke
result = graph.invoke({
    "messages": [HumanMessage(content="What is RAG?")],
    "user_id": "user-1",
    "context": None,
    "iterations": 0
})
print(result["messages"][-1].content)

# Stream — yields state updates after each node
for chunk in graph.stream({"messages": [HumanMessage(content="Hello")]}, stream_mode="updates"):
    print(chunk)
```

### 4. Conditional Edges (Routing)

The key to branching logic — use a function that reads the state and returns the name of the next node.

```python
from langchain_core.messages import ToolMessage

def should_continue(state: AgentState) -> str:
    """Router: return the name of the next node."""
    last_message = state["messages"][-1]

    # If the LLM called a tool, go to tool executor
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"

    # Otherwise, we're done
    return END

def check_iterations(state: AgentState) -> str:
    if state["iterations"] >= 5:
        return "max_reached"
    return "continue"

builder.add_conditional_edges(
    "llm",                    # from this node
    should_continue,          # call this function
    {                         # map return values to node names
        "tools": "tools",
        END: END
    }
)

# Shorthand when keys match values
builder.add_conditional_edges("llm", should_continue)
```

### 5. Tool-Calling Agent (ReAct Pattern)

The most common LangGraph pattern: LLM → decide to call tools → execute tools → LLM → ...

```python
from langchain_core.tools import tool
from langgraph.prebuilt import ToolNode

@tool
def search_web(query: str) -> str:
    """Search the web for current information."""
    # your search implementation
    return f"Results for: {query}"

@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    return f"Sunny, 25°C in {city}"

tools = [search_web, get_weather]
llm_with_tools = llm.bind_tools(tools)

def call_llm(state: AgentState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    last = state["messages"][-1]
    if last.tool_calls:
        return "tools"
    return END

# ToolNode handles tool execution automatically
tool_node = ToolNode(tools)

builder = StateGraph(AgentState)
builder.add_node("llm", call_llm)
builder.add_node("tools", tool_node)

builder.add_edge(START, "llm")
builder.add_conditional_edges("llm", should_continue)
builder.add_edge("tools", "llm")   # loop back to LLM after tools

graph = builder.compile()
```

### 6. Human-in-the-Loop

Interrupt the graph at specific nodes and wait for human input before continuing.

```python
from langgraph.checkpoint.memory import MemorySaver

# Use a checkpointer to save state between interruptions
checkpointer = MemorySaver()

graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["risky_action"]   # pause before this node
)

config = {"configurable": {"thread_id": "session-1"}}  # unique per conversation

# First run — pauses before "risky_action"
result = graph.invoke(initial_state, config=config)
print("Graph paused. Review the state:")
print(result)

# Get the current state (to show the human)
current_state = graph.get_state(config)
print(current_state.values)

# Resume after human approval
graph.invoke(None, config=config)  # None = resume from where it stopped

# Or update state before resuming (inject human feedback)
graph.update_state(config, {"messages": [HumanMessage("Approved, proceed.")]})
graph.invoke(None, config=config)
```

### 7. Persistence (Cross-Session Memory)

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# SQLite — great for dev and single-instance prod
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "user-123"}}

    # First message
    graph.invoke({"messages": [HumanMessage("My name is Joshua")]}, config=config)

    # Second message in a new Python session — state is restored automatically
    graph.invoke({"messages": [HumanMessage("What's my name?")]}, config=config)
    # LLM sees full history → answers "Joshua"
```

### 8. Multi-Agent Subgraphs

```python
# Each agent is a compiled graph
researcher_graph = build_researcher()
writer_graph = build_writer()

# Orchestrator graph calls subgraphs as nodes
def run_researcher(state: OrchestratorState) -> dict:
    result = researcher_graph.invoke({"messages": state["messages"]})
    return {"research": result["messages"][-1].content}

def run_writer(state: OrchestratorState) -> dict:
    result = writer_graph.invoke({
        "messages": state["messages"],
        "context": state["research"]
    })
    return {"draft": result["messages"][-1].content}

orchestrator = StateGraph(OrchestratorState)
orchestrator.add_node("researcher", run_researcher)
orchestrator.add_node("writer", run_writer)
orchestrator.add_edge(START, "researcher")
orchestrator.add_edge("researcher", "writer")
orchestrator.add_edge("writer", END)
```

### 9. Streaming

```python
# Stream node outputs as they complete
async for chunk in graph.astream(
    {"messages": [HumanMessage("Explain RAG")]},
    stream_mode="updates"
):
    for node_name, node_output in chunk.items():
        print(f"Node: {node_name}")
        if "messages" in node_output:
            print(node_output["messages"][-1].content)

# Stream LLM token by token (values mode)
async for chunk in graph.astream(
    {"messages": [HumanMessage("Explain RAG")]},
    stream_mode="messages"
):
    if chunk[1]["langgraph_node"] == "llm":
        print(chunk[0].content, end="", flush=True)
```

---

## Most-Used Patterns

### State with Reducers (Custom Merge Logic)

```python
import operator

class State(TypedDict):
    messages: Annotated[list, add_messages]   # append
    scores:   Annotated[list, operator.add]   # also append
    count:    Annotated[int, operator.add]    # sum (not replace)
    status:   str                             # replace (no reducer)
```

### Parallel Execution

```python
# Multiple nodes can run in parallel if they have no dependency on each other
builder.add_edge("start", "node_a")
builder.add_edge("start", "node_b")   # runs in parallel with node_a
builder.add_edge("node_a", "merge")
builder.add_edge("node_b", "merge")
```

---

## Gotchas

- **State updates are merged, not replaced** — returning `{"messages": [...]}` from a node merges into state, not overwrites, unless you define a custom reducer.
- **`add_messages` deduplicates by ID** — if you return a message with the same ID twice, it updates instead of duplicating.
- **Cycles need a termination condition** — always have a conditional edge that can route to `END`, or the graph runs forever.
- **`interrupt_before` requires a checkpointer** — without one, the graph has no memory to resume from.
- **`thread_id` is mandatory with checkpointers** — always pass `config={"configurable": {"thread_id": "..."}}` when using persistence.
- **Async graphs need async nodes** — if you compile an async graph, all nodes should be `async def`. Mixing sync and async can cause issues.

---

## Quick Links

- [LangGraph Docs](https://langchain-ai.github.io/langgraph/)
- [LangGraph How-to Guides](https://langchain-ai.github.io/langgraph/how-tos/)
- [LangGraph Tutorials](https://langchain-ai.github.io/langgraph/tutorials/)
- [LangSmith](https://smith.langchain.com) — trace and debug graph runs
- [langgraph-checkpoint-postgres](https://pypi.org/project/langgraph-checkpoint-postgres/) — production persistence
