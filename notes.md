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
def breakpoint(state: State) -> State:characters
  if len(state['input']) > 5:
    raise NodeInterrupt(f"Received input that is longer than 5 characters: {state['input']}")

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

## Stream

```py
# via SDK
async for event in graph.astream_events({"messages": [input_message]}, config, version="v2"):
    # Get chat model tokens from a particular node
    if event["event"] == "on_chat_model_stream" and event['metadata'].get('langgraph_node','') == node_to_stream:
        data = event["data"]
        print(data["chunk"].content, end="|")

# via API
async for event in client.runs.stream(thread["thread_id"], assistant_id="agent", input={"messages": [input_message]}, stream_mode="values"):
    messages = event.data.get('messages',None)
    if messages:
        print(convert_to_messages(messages)[-1])
    print('='*25)
```

## Interrupt

```py
# Interrupt
graph = builder.compile(interrupt_before=["tools"], checkpointer=memory)

# Continue
## SDK
for event in graph.stream(None, thread, stream_mode="values"):
## API
async for chunk in client.runs.stream(
    thread["thread_id"],
    "agent",
    input=None,
    stream_mode="values",
    interrupt_before=["tools"],
): ...

# Human in the Loop
def human_feedback(state: MessagesState):
    pass
graph = builder.compile(interrupt_before=["human_feedback"], checkpointer=memory)

for event in graph.stream(initial_input, thread, stream_mode="values"):
    event["messages"][-1].pretty_print()

user_input = input("Tell me how you want to update the state: ")
graph.update_state(thread, {"messages": user_input}, as_node="human_feedback")

for event in graph.stream(None, thread, stream_mode="values"):
    event["messages"][-1].pretty_print()
```

## States

```py
## SDK
state = graph.get_state(thread)
graph.update_state(thread,{"messages": [HumanMessage(content="No, actually multiply 3 and 3!")]})
### Time Travel / Replay
all_states = [s for s in graph.get_state_history(thread)]
for event in graph.stream(None, all_states[-2].config, stream_mode="values"):
    event['messages'][-1].pretty_print()
### Fork
fork_config = graph.update_state(
    all_states[-2].config,
    {"messages": [HumanMessage(content='Multiply 5 and 3',
                               id=to_fork.values["messages"][0].id)]},
)
for event in graph.stream(None, fork_config, stream_mode="values"):
    event['messages'][-1].pretty_print()

## API
current_state = await client.threads.get_state(thread['thread_id'])
last_message = current_state['values']['messages'][-1]
await client.threads.update_state(thread['thread_id'], {"messages": last_message})
async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id="agent",
    input=None,
    stream_mode="values",
    interrupt_before=["assistant"],
): ...
### Time Travel / Replay
states = await client.threads.get_history(thread['thread_id'])
async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id="agent",
    input=None,
    stream_mode="updates",
    checkpoint_id=states[-2]['checkpoint_id']
):
### Fork
forked_config = await client.threads.update_state(
    thread["thread_id"],
    {"messages": HumanMessage(content="Multiply 3 and 3",
      id=to_fork['values']['messages'][0]['id'])},
    checkpoint_id=states[-2]['checkpoint_id']
)
async for chunk in client.runs.stream(
    thread["thread_id"],
    assistant_id="agent",
    input=None,
    stream_mode="updates",
    checkpoint_id=forked_config['checkpoint_id']
): ...
```
