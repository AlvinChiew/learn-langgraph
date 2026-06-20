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

# Add sub-graph
builder.add_node("question_summarization", sub_builder.compile())
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

## Parallelism - Map Reduce

- Branch output to multiple nodes

```py
from operator import add
from typing import Annotated
from langgraph.types import Send


class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    contents: Annotated[list[int], add]
    best_content: str

class SubjectState(TypedDict):
    subject: str

class ContentState(BaseModel):
    content: Annotated[list[int], add]

# Map
def distribute(state: OverallState):
    return [Send("node1", {"subject": s}) for s in state["subjects"]]

def generate_content(state: SubjectState):
    ...
    return {"contents": [response.content]}

# Reduce
def get_best_content(state: OverallState):
    ...
    return {"best_content": state["contents"][response.id]}

graph = StateGraph(OverallState)
graph.add_node("generate_subjects", generate_subjects)
graph.add_node("generate_content", generate_content)
graph.add_node("get_best_content", get_best_content)

graph.add_edge(START, "generate_subjects")
graph.add_conditional_edges("generate_subjects", distribute, ["generate_content"])
graph.add_edge("generate_content", "get_best_content")
graph.add_edge("get_best_content", END)

app = graph.compile()
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

## Long-Term Memory

```py
import uuid
from datetime import datetime
from pydantic import BaseModel, Field
from typing import Literal, Optional, TypedDict

from trustcall import create_extractor

from langchain_core.runnables import RunnableConfig
from langchain_core.messages import merge_message_runs
from langchain_core.messages import SystemMessage, HumanMessage
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore

import configuration

model = ChatOpenAI(model="gpt-4o", temperature=0)

# Procedural (Instructions) - System Prompt
MODEL_SYSTEM_MESSAGE = "... {user_profile} ... {todo} ... {instructions} ..."

# Semantic (Facts) - User profile schema
class Profile(BaseModel):
    """This is the profile of the user you are chatting with"""
    name: Optional[str] = Field(description="The user's name", default=None)
    interests: list[str] = Field(
        description="Interests that the user has",
        default_factory=list
    )

# Episodic (Memories) - ToDo schema
class ToDo(BaseModel):
    task: str = Field(description="The task to be completed.")
    deadline: Optional[datetime] = Field(
        description="When the task needs to be completed by (if applicable)",
        default=None
    )
    status: Literal["not started", "in progress", "done", "archived"] = Field(
        description="Current status of the task",
        default="not started"
    )

class MemoryType(TypedDict):
    """ Decision on what memory type to update """
    memory_type: Literal['user', 'todo', 'instructions']



def memory_assistant(state: MessagesState, config: RunnableConfig, store: BaseStore):
    """Load memories from the store and use them to personalize the chatbot's response."""

    # Get the user ID from the config
    ...

    namespace = ("profile", user_id)
    memories = store.search(namespace)
    user_profile = memories[0].value if memories else None

    namespace = ("todo", user_id)
    memories = store.search(namespace)
    todo = "\n".join(f"{mem.value}" for mem in memories)

    namespace = ("instructions", user_id)
    memories = store.search(namespace)
    instructions = memories[0].value if memories else ""

    system_msg = MODEL_SYSTEM_MESSAGE.format(user_profile=user_profile, todo=todo, instructions=instructions)
    response = model.bind_tools([MemoryType], parallel_tool_calls=False).invoke([SystemMessage(content=system_msg)]+state["messages"])
    return {"messages": [response]}

def update_profile(state: MessagesState, config: RunnableConfig, store: BaseStore):
    """Reflect on the chat history and update the memory collection."""

    # Get the user ID from the config
    ...

    namespace = ("profile", user_id)
    existing_items = store.search(namespace)
    tool_name = "Profile"
    existing_memories = ([(existing_item.key, tool_name, existing_item.value)
                          for existing_item in existing_items]
                          if existing_items
                          else None
                        )

    TRUSTCALL_INSTRUCTION_FORMATTED=TRUSTCALL_INSTRUCTION.format(time=datetime.now().isoformat())
    updated_messages=list(merge_message_runs(messages=[SystemMessage(content=TRUSTCALL_INSTRUCTION_FORMATTED)] + state["messages"][:-1]))

    profile_extractor = create_extractor(
        model,
        tools=[Profile],
        tool_choice="Profile",
    )

    result = profile_extractor.invoke({"messages": updated_messages,
                                         "existing": existing_memories})

    for r, rmeta in zip(result["responses"], result["response_metadata"]):
        store.put(namespace,
                  rmeta.get("json_doc_id", str(uuid.uuid4())),
                  r.model_dump(mode="json"),
            )
    tool_calls = state['messages'][-1].tool_calls
    return {"messages": [{"role": "tool", "content": "updated profile", "tool_call_id":tool_calls[0]['id']}]}

def update_todos(state: MessagesState, config: RunnableConfig, store: BaseStore):

    """Reflect on the chat history and update the memory collection."""

    # Get the user ID from the config
    ...

    namespace = ("todo", user_id)
    existing_items = store.search(namespace)
    tool_name = "ToDo"
    existing_memories = ([(existing_item.key, tool_name, existing_item.value)
                          for existing_item in existing_items]
                          if existing_items
                          else None
                        )

    TRUSTCALL_INSTRUCTION_FORMATTED=TRUSTCALL_INSTRUCTION.format(time=datetime.now().isoformat())
    updated_messages=list(merge_message_runs(messages=[SystemMessage(content=TRUSTCALL_INSTRUCTION_FORMATTED)] + state["messages"][:-1]))

    todo_extractor = create_extractor(
        model,
        tools=[ToDo],
        tool_choice=tool_name,
        enable_inserts=True
    ).with_listeners(on_end=spy)

    result = todo_extractor.invoke({"messages": updated_messages,
                                    "existing": existing_memories})
    for r, rmeta in zip(result["responses"], result["response_metadata"]):
        store.put(namespace,
                  rmeta.get("json_doc_id", str(uuid.uuid4())),
                  r.model_dump(mode="json"),
            )

    tool_calls = state['messages'][-1].tool_calls
    todo_update_msg = extract_tool_info(spy.called_tools, tool_name)
    return {"messages": [{"role": "tool", "content": todo_update_msg, "tool_call_id":tool_calls[0]['id']}]}

def update_instructions(state: MessagesState, config: RunnableConfig, store: BaseStore):
    """Reflect on the chat history and update the memory collection."""

    # Get the user ID from the config
    ...

    namespace = ("instructions", user_id)

    existing_memory = store.get(namespace, "user_instructions")

    # Format the memory in the system prompt
    system_msg = CREATE_INSTRUCTIONS.format(current_instructions=existing_memory.value if existing_memory else None)
    new_memory = model.invoke([SystemMessage(content=system_msg)]+state['messages'][:-1] + [HumanMessage(content="Please update the instructions based on the conversation")])

    key = "user_instructions"
    store.put(namespace, key, {"memory": new_memory.content})
    tool_calls = state['messages'][-1].tool_calls
    return {"messages": [{"role": "tool", "content": "updated instructions", "tool_call_id":tool_calls[0]['id']}]}

def route_message(state: MessagesState, config: RunnableConfig, store: BaseStore) -> Literal[END, "update_todos", "update_instructions", "update_profile"]:
    """Reflect on the memories and chat history to decide whether to update the memory collection."""
    message = state['messages'][-1]
    if len(message.tool_calls) ==0:
        return END
    else:
        tool_call = message.tool_calls[0]
        if tool_call['args']['memory_type'] == "user":
            return "update_profile"
        elif tool_call['args']['memory_type'] == "todo":
            return "update_todos"
        elif tool_call['args']['memory_type'] == "instructions":
            return "update_instructions"
        else:
            raise ValueError

# Create the graph + all nodes
builder = StateGraph(MessagesState, config_schema=configuration.Configuration)

# Define the flow of the memory extraction process
builder.add_node(memory_assistant)
builder.add_node(update_todos)
builder.add_node(update_profile)
builder.add_node(update_instructions)

# Define the flow
builder.add_edge(START, "memory_assistant")
builder.add_conditional_edges("memory_assistant", route_message)
builder.add_edge("update_todos", "memory_assistant")
builder.add_edge("update_profile", "memory_assistant")
builder.add_edge("update_instructions", "memory_assistant")

# Compile the graph
graph = builder.compile()
```
