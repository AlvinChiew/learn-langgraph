# Notes

## Snippets

```py
from langchain_core.messages import SystemMessage
from langchain_openai import ChatOpenAI

from langgraph.graph import START, StateGraph, MessagesState
from langgraph.prebuilt import tools_condition, ToolNode
from langgraph.checkpoint.memory import MemorySaver


# Tool
def add(a: int, b: int) -> int:
  """Adds a and b.

  Args:
    a: first int
    b: second int
  """
  return a + b


# LLM
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([add])

# Memory
memory = MemorySaver()

# System prompt
sys_msg = SystemMessage(content="You are a helpful assistant tasked with writing performing arithmetic on a set of inputs.")

# Graph
builder = StateGraph(MessagesState)
builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes  to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
builder.add_edge("tools", "assistant")

# Compile graph
graph = builder.compile(checkpointer=memory)



# Invoke
config = {"configurable": {"thread_id": "1"}}   # Short term memory
messages = [HumanMessage(content="Add 3 and 4.")]
messages = graph.invoke({"messages": messages},config)
for m in messages['messages']:
    m.pretty_print()

messages = [HumanMessage(content="Add that by 2.")]
messages = react_graph_memory.invoke({"messages": messages}, config)
for m in messages['messages']:
    m.pretty_print()
```

## Host

```sh
langgraph dev
```

## Consume

```py
from langgraph_sdk import get_client
from langchain_core.messages import HumanMessage


URL = "http://127.0.0.1:2024"
client = get_client(url=URL)
assistants = await client.assistants.search()[0]
thread = await client.threads.create()

input = {"messages": [HumanMessage(content="Add 3 and 2.")]}
async for chunk in client.runs.stream(
    thread['thread_id'],
    "agent",
    input=input,
    stream_mode="values",
  ):
  if chunk.data and chunk.event != "metadata":
    print(chunk.data['messages'][-1])
```
