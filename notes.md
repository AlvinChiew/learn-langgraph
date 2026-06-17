# Notes

## Snippets

```py
from langchain_core.messages import SystemMessage
from langchain_core.messages import RemoveMessage
from langchain_core.messages import trim_messages
from langchain_openai import ChatOpenAI

from langgraph.graph import START, StateGraph, MessagesState
from langgraph.prebuilt import tools_condition, ToolNode
from langgraph.checkpoint.memory import MemorySaver


# LLM
llm = ChatOpenAI(model="gpt-4o")
llm_with_tools = llm.bind_tools([add])

# Memory
memory = MemorySaver()

# System prompt
sys_msg = SystemMessage(content="You are a helpful assistant tasked with writing performing arithmetic on a set of inputs.")


# Custom Message State
class State(MessagesState):
  summary: str

# Tool
def add(a: int, b: int) -> int:
  """Adds a and b.

  Args:
    a: first int
    b: second int
  """
  return a + b

# Node
def need_summarize(state: State) -> Literal["summarize_conversation", "__end__"]:
  """Return the next node to execute."""
  messages = state["messages"]
  if len(messages) > 6:
    return "summarize_conversation"
  return END

def summarize_conversation(state: State):
  summary = state.get("summary", "")
  if summary:
    summary_message = (
      f"This is summary of the conversation to date: {summary}\n\n"
      "Extend the summary by taking into account the new messages above:"
    )
  else:
    summary_message = "Create a summary of the conversation above:"

  messages = state["messages"] + [HumanMessage(content=summary_message)]
  response = model.invoke(messages)

  delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
  return {"summary": response.content, "messages": delete_messages}

def assistant(state: MessagesState):
  messages = trim_messages(
          state["messages"],
          max_tokens=100,
          strategy="last",
          token_counter=ChatOpenAI(model="gpt-4o"),
          allow_partial=False,
      )
  return {"messages": [llm_with_tools.invoke(messages)]}


# Graph
builder = StateGraph(MessagesState)

builder.add_node("assistant", assistant)
builder.add_node("tools", ToolNode([add]))
workflow.add_node(summarize_conversation)

builder.add_edge(START, "assistant")
builder.add_conditional_edges(
    "assistant",
    # If the latest message (result) from assistant is a tool call -> tools_condition routes  to tools
    # If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
    tools_condition,
)
workflow.add_conditional_edges("assistant", need_summarize)
workflow.add_edge("summarize_conversation", END)

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


# Deterministic condition
def decide_mood(state) -> Literal["node_2", "node_3"]:
  user_input = state['graph_state']
  if random.random() < 0.5:
    return "node_2"
  return "node_3"

builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)
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

## Data Class

### Simple

```py
from typing import Literal
from dataclasses import dataclass

@dataclass
class MoodState:
    name: str
    mood: Literal["happy","sad"]
```

### With Validator

```py
from pydantic import BaseModel, field_validator, ValidationError

class MoodState(BaseModel):
    name: str
    mood: str # "happy" or "sad"

    @field_validator('mood')
    @classmethod
    def validate_mood(cls, value):
        # Ensure the mood is either "happy" or "sad"
        if value not in ["happy", "sad"]:
            raise ValueError("Each mood must be either 'happy' or 'sad'")
        return value

try:
    state = MoodState(name="John Doe", mood="mad")
except ValidationError as e:
    print("Validation Error:", e)
```

## Reducer

- When an output is branched to multiple nodes

```py
from operator import add
from typing import Annotated

class State(TypedDict):
    foo: Annotated[list[int], add]

def node_1(state):
    return {"foo": [state['foo'][0] + 1]}

def node_2(state):
    return {"foo": [state['foo'][-1] + 1]}
```
