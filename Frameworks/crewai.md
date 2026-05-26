# CrewAI Cheatsheet

## Mental Model

CrewAI is a framework for orchestrating **role-based multi-agent systems**. The mental model maps directly to a team of people: you define **Agents** (specialists with a role, goal, and backstory), **Tasks** (concrete pieces of work with expected outputs), and a **Crew** (the team + the process that coordinates them). CrewAI handles the orchestration loop — assigning tasks, passing outputs between agents, and managing tool calls. The abstraction level is higher than LangGraph: you describe *what* agents should do, not *how* the graph flows.

**CrewAI vs LangGraph:**
- CrewAI: high-level, role-based, less control, faster to prototype
- LangGraph: low-level, graph-based, full control over flow, better for complex state management

---

## Install & Minimal Setup

```bash
pip install crewai crewai-tools
pip install 'crewai[tools]'   # includes common tools

# Set your LLM API key
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

---

## Core Concepts

### 1. Agent

An autonomous unit with a role, goal, and backstory. The backstory acts as a system prompt that shapes behavior.

```python
from crewai import Agent
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Default uses OpenAI — configure with llm param to use others
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0.2)

researcher = Agent(
    role="Senior AI Research Analyst",
    goal="Find and synthesize the most relevant and up-to-date information on {topic}",
    backstory="""You are a meticulous researcher with 10 years of experience in AI.
    You know how to find credible sources, identify key insights, and ignore noise.
    You always cite your sources and flag uncertainty.""",
    llm=llm,
    tools=[],                # tools this agent can use
    verbose=True,            # print agent reasoning
    allow_delegation=False,  # can this agent delegate to others?
    max_iter=5,              # max reasoning iterations before forced output
    memory=True,             # enable short-term memory within a crew run
)

writer = Agent(
    role="Technical Content Writer",
    goal="Write clear, engaging, and accurate technical content based on research provided",
    backstory="""You are an expert technical writer who translates complex AI concepts
    into accessible content. You write with precision, use concrete examples,
    and structure content for maximum clarity.""",
    llm=llm,
    verbose=True,
)
```

### 2. Task

A specific, concrete piece of work. The `description` tells the agent what to do; `expected_output` defines the format and quality bar.

```python
from crewai import Task

research_task = Task(
    description="""Research the current state of {topic}.
    Focus on:
    1. Key techniques and approaches
    2. Recent developments in the last 6 months
    3. Practical applications and limitations
    4. Leading tools and frameworks

    Use the search tool to find current information.""",

    expected_output="""A structured research report with:
    - Executive summary (3-5 sentences)
    - Key findings (bullet points)
    - Recent developments (with dates)
    - Practical applications
    - Limitations and open problems
    - Sources cited""",

    agent=researcher,    # agent responsible for this task
    tools=[],            # override agent tools for this task (optional)
    output_file="research.md",  # save output to file (optional)
    async_execution=False       # True to run in parallel with other async tasks
)

writing_task = Task(
    description="""Based on the research provided, write a comprehensive technical blog post
    about {topic} for an audience of senior software engineers.

    The post should:
    - Start with a compelling hook
    - Explain the core concepts clearly
    - Include practical code examples where relevant
    - End with actionable takeaways
    - Be 1500-2000 words""",

    expected_output="""A complete, publication-ready blog post in Markdown format
    with proper headings, code blocks, and a conclusion.""",

    agent=writer,
    context=[research_task],   # this task gets research_task's output as context
    output_file="post.md"
)
```

### 3. Tools

Tools extend what agents can do — web search, file operations, API calls, etc.

```python
from crewai_tools import (
    SerperDevTool,       # web search via Serper API
    FileReadTool,
    DirectoryReadTool,
    WebsiteSearchTool,
    YoutubeVideoSearchTool,
    GithubSearchTool,
)
from langchain_core.tools import tool

# Built-in tools
search_tool = SerperDevTool()
file_tool = FileReadTool()

# Custom tool — use @tool decorator
@tool("Database Query Tool")
def query_db(sql: str) -> str:
    """Execute a SQL query and return results as a string. Use for data retrieval."""
    result = db.execute(sql)
    return str(result.fetchall())

# Assign to agents
analyst = Agent(
    role="Data Analyst",
    goal="Analyze data to answer business questions",
    backstory="Expert data analyst with strong SQL skills.",
    tools=[search_tool, query_db]
)
```

### 4. Crew & Process

The Crew assembles agents and tasks, defines the execution process, and runs the workflow.

```python
from crewai import Crew, Process

# Sequential process — tasks run one after another in order
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,   # default
    verbose=True,                 # print crew activity
    memory=True,                  # shared memory across agents
    max_rpm=10,                   # rate limit: requests per minute to LLM
    share_crew=False,             # don't share crew data with CrewAI
)

# Hierarchical process — a manager agent delegates tasks
manager = Agent(
    role="Project Manager",
    goal="Coordinate the team to deliver high-quality research and content",
    backstory="Experienced project manager who delegates effectively.",
    llm=llm,
    allow_delegation=True   # required for manager role
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.hierarchical,
    manager_agent=manager,    # or manager_llm=llm (auto-creates manager)
    verbose=True,
)

# Kickoff — run the crew
result = crew.kickoff(inputs={"topic": "Retrieval-Augmented Generation"})
print(result.raw)
```

### 5. Crew Kickoff Variants

```python
# Standard — synchronous, returns final output
result = crew.kickoff(inputs={"topic": "RAG"})

# Async
result = await crew.kickoff_async(inputs={"topic": "RAG"})

# For each — run the same crew on a list of inputs in parallel
inputs_list = [
    {"topic": "RAG"},
    {"topic": "Fine-tuning"},
    {"topic": "Prompt Engineering"},
]
results = crew.kickoff_for_each(inputs=inputs_list)

# Async for each
results = await crew.kickoff_for_each_async(inputs=inputs_list)
```

### 6. Memory

```python
# Enable all memory types in the Crew
crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True,
    # memory_config can fine-tune which memories are active
    memory_config={
        "provider": "mem0",     # or "basic" (default in-memory)
    }
)
```

| Memory type | Scope | What it stores |
|---|---|---|
| **Short-term** | Within a run | Recent interactions in the current crew run |
| **Long-term** | Across runs | Key facts from past runs (persisted to SQLite) |
| **Entity** | Across runs | People, orgs, and concepts encountered |
| **Contextual** | Per task | Task context assembled from all memory types |

### 7. Flows (Conditional Orchestration — CrewAI 0.70+)

For more complex orchestration with state and branching — closer to LangGraph but with CrewAI's syntax.

```python
from crewai.flow.flow import Flow, listen, start, router

class ContentFlow(Flow):
    @start()
    def classify_request(self):
        # First step — always runs
        self.state["type"] = classify(self.state["input"])
        return self.state["type"]

    @router(classify_request)
    def route(self):
        if self.state["type"] == "research":
            return "do_research"
        return "do_writing"

    @listen("do_research")
    def do_research(self):
        result = research_crew.kickoff(inputs={"topic": self.state["input"]})
        self.state["output"] = result.raw

    @listen("do_writing")
    def do_writing(self):
        result = writing_crew.kickoff(inputs={"prompt": self.state["input"]})
        self.state["output"] = result.raw

flow = ContentFlow()
result = flow.kickoff(inputs={"input": "Explain transformers"})
```

---

## Most-Used Patterns

### Input Variables

Use `{variable}` placeholders in `description`, `goal`, and `expected_output` — they're filled in at `kickoff(inputs={...})`.

```python
task = Task(
    description="Analyze the {data_source} dataset for {metric} anomalies in {time_period}.",
    expected_output="Anomaly report for {metric} covering {time_period}.",
    agent=analyst
)

crew.kickoff(inputs={
    "data_source": "sales",
    "metric": "revenue",
    "time_period": "Q4 2024"
})
```

### Output Pydantic Models

```python
from pydantic import BaseModel

class ResearchOutput(BaseModel):
    summary: str
    key_findings: list[str]
    sources: list[str]

research_task = Task(
    description="Research {topic}",
    expected_output="Structured research findings",
    agent=researcher,
    output_pydantic=ResearchOutput   # forces structured output
)

result = crew.kickoff(inputs={"topic": "RAG"})
output: ResearchOutput = result.pydantic
print(output.key_findings)
```

---

## Gotchas

- **`allow_delegation=True` adds latency** — agents with delegation can spawn sub-tasks and call other agents. Only enable it for manager-type agents.
- **`context` creates task dependencies** — if task B has `context=[task_a]`, task B waits for task A to finish and receives its output. This is sequential even in a "parallel" crew.
- **Verbose output can be overwhelming** — set `verbose=False` in production; use `verbose=True` only for debugging.
- **Tool descriptions matter** — agents choose tools based on the `description` in the `@tool` decorator. Write clear, specific descriptions or agents will misuse tools.
- **`max_iter` prevents infinite loops** — if an agent doesn't produce the `expected_output` within `max_iter` attempts, it returns what it has. Increase it for complex tasks.
- **Memory costs tokens** — enabling memory injects past interactions into every LLM call. Monitor token usage if cost is a concern.
- **Hierarchical process needs `allow_delegation=True`** — if your manager agent can't delegate, hierarchical mode silently falls back to sequential.

---

## Quick Links

- [CrewAI Docs](https://docs.crewai.com)
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI)
- [crewai-tools](https://github.com/crewAIInc/crewAI-tools) — built-in tool catalog
- [CrewAI Examples](https://github.com/crewAIInc/crewAI-examples)
- [CrewAI vs LangGraph comparison](https://docs.crewai.com/concepts/langgraph-vs-crewai)
