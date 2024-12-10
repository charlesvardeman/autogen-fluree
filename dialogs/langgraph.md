Skip to content

LangGraph Glossary

Initializing search

GitHub

  * Home 
  * Tutorials 
  * How-to Guides 
  * Conceptual Guides 
  * Reference 

GitHub

  * Home 
  * Tutorials 
  * How-to Guides 
  * Conceptual Guides 

Conceptual Guides

    * LangGraph  LangGraph 
      * LangGraph 
      * Why LangGraph? 
      * LangGraph Glossary  LangGraph Glossary  Table of contents 
        * Graphs 
          * StateGraph 
          * MessageGraph 
          * Compiling your graph 
        * State 
          * Schema 
            * Multiple schemas 
          * Reducers 
            * Default Reducer 
          * Working with Messages in Graph State 
            * Why use messages? 
            * Using Messages in your Graph 
            * Serialization 
            * MessagesState 
        * Nodes 
          * START Node 
          * END Node 
        * Edges 
          * Normal Edges 
          * Conditional Edges 
          * Entry Point 
          * Conditional Entry Point 
        * Send 
        * Command 
          * Using inside tools 
        * Persistence 
        * Threads 
        * Storage 
        * Graph Migrations 
        * Configuration 
          * Recursion Limit 
        * Breakpoints 
          * Dynamic Breakpoints 
        * Subgraphs 
          * As a compiled graph 
          * As a function 
        * Visualization 
        * Streaming 
      * Agent architectures 
      * Multi-agent Systems 
      * Human-in-the-loop 
      * Persistence 
      * Memory 
      * Streaming 
      * FAQ 
    * LangGraph Platform  LangGraph Platform 
      * LangGraph Platform 
      * High Level 
      * Components 
      * LangGraph Server 
      * Deployment Options 
  * Reference 

Table of contents

  * Graphs 
    * StateGraph 
    * MessageGraph 
    * Compiling your graph 
  * State 
    * Schema 
      * Multiple schemas 
    * Reducers 
      * Default Reducer 
    * Working with Messages in Graph State 
      * Why use messages? 
      * Using Messages in your Graph 
      * Serialization 
      * MessagesState 
  * Nodes 
    * START Node 
    * END Node 
  * Edges 
    * Normal Edges 
    * Conditional Edges 
    * Entry Point 
    * Conditional Entry Point 
  * Send 
  * Command 
    * Using inside tools 
  * Persistence 
  * Threads 
  * Storage 
  * Graph Migrations 
  * Configuration 
    * Recursion Limit 
  * Breakpoints 
    * Dynamic Breakpoints 
  * Subgraphs 
    * As a compiled graph 
    * As a function 
  * Visualization 
  * Streaming 

  1. Home 
  2. Conceptual Guides 
  3. LangGraph 

# LangGraph Glossary¶

## Graphs¶

At its core, LangGraph models agent workflows as graphs. You define the behavior of your agents using three key components:

  1. `State`: A shared data structure that represents the current snapshot of your application. It can be any Python type, but is typically a `TypedDict` or Pydantic `BaseModel`.

  2. `Nodes`: Python functions that encode the logic of your agents. They receive the current `State` as input, perform some computation or side-effect, and return an updated `State`.

  3. `Edges`: Python functions that determine which `Node` to execute next based on the current `State`. They can be conditional branches or fixed transitions.

By composing `Nodes` and `Edges`, you can create complex, looping workflows that evolve the `State` over time. The real power, though, comes from how LangGraph manages that `State`. To emphasize: `Nodes` and `Edges` are nothing more than Python functions - they can contain an LLM or just good ol' Python code.

In short: _nodes do the work. edges tell what to do next_.

LangGraph's underlying graph algorithm uses message passing to define a general program. When a Node completes its operation, it sends messages along one or more edges to other node(s). These recipient nodes then execute their functions, pass the resulting messages to the next set of nodes, and the process continues. Inspired by Google's Pregel system, the program proceeds in discrete "super-steps."

A super-step can be considered a single iteration over the graph nodes. Nodes that run in parallel are part of the same super-step, while nodes that run sequentially belong to separate super-steps. At the start of graph execution, all nodes begin in an `inactive` state. A node becomes `active` when it receives a new message (state) on any of its incoming edges (or "channels"). The active node then runs its function and responds with updates. At the end of each super-step, nodes with no incoming messages vote to `halt` by marking themselves as `inactive`. The graph execution terminates when all nodes are `inactive` and no messages are in transit.

### StateGraph¶

The `StateGraph` class is the main graph class to use. This is parameterized by a user defined `State` object.

### MessageGraph¶

The `MessageGraph` class is a special type of graph. The `State` of a `MessageGraph` is ONLY a list of messages. This class is rarely used except for chatbots, as most applications require the `State` to be more complex than a list of messages.

### Compiling your graph¶

To build your graph, you first define the state, you then add nodes and edges, and then you compile it. What exactly is compiling your graph and why is it needed?

Compiling is a pretty simple step. It provides a few basic checks on the structure of your graph (no orphaned nodes, etc). It is also where you can specify runtime args like checkpointers and breakpoints. You compile your graph by just calling the `.compile` method:

```
graph = graph_builder.compile(...)

```

You **MUST** compile your graph before you can use it.

## State¶

The first thing you do when you define a graph is define the `State` of the graph. The `State` consists of the schema of the graph as well as `reducer` functions which specify how to apply updates to the state. The schema of the `State` will be the input schema to all `Nodes` and `Edges` in the graph, and can be either a `TypedDict` or a `Pydantic` model. All `Nodes` will emit updates to the `State` which are then applied using the specified `reducer` function.

### Schema¶

The main documented way to specify the schema of a graph is by using `TypedDict`. However, we also support using a Pydantic BaseModel as your graph state to add **default values** and additional data validation.

By default, the graph will have the same input and output schemas. If you want to change this, you can also specify explicit input and output schemas directly. This is useful when you have a lot of keys, and some are explicitly for input and others for output. See the notebook here for how to use.

#### Multiple schemas¶

Typically, all graph nodes communicate with a single schema. This means that they will read and write to the same state channels. But, there are cases where we want more control over this:

  * Internal nodes can pass information that is not required in the graph's input / output.
  * We may also want to use different input / output schemas for the graph. The output might, for example, only contain a single relevant output key.

It is possible to have nodes write to private state channels inside the graph for internal node communication. We can simply define a private schema, `PrivateState`. See this notebook for more detail.

It is also possible to define explicit input and output schemas for a graph. In these cases, we define an "internal" schema that contains _all_ keys relevant to graph operations. But, we also define `input` and `output` schemas that are sub-sets of the "internal" schema to constrain the input and output of the graph. See this notebook for more detail.

Let's look at an example:

```
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # Write to OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # Read from OverallState, write to PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # Read from PrivateState, write to OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input=InputState,output=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
{'graph_output': 'My name is Lance'}

```

There are two subtle and important points to note here:

  1. We pass `state: InputState` as the input schema to `node_1`. But, we write out to `foo`, a channel in `OverallState`. How can we write out to a state channel that is not included in the input schema? This is because a node _can write to any state channel in the graph state._ The graph state is the union of of the state channels defined at initialization, which includes `OverallState` and the filters `InputState` and `OutputState`.

  2. We initialize the graph with `StateGraph(OverallState,input=InputState,output=OutputState)`. So, how can we write to `PrivateState` in `node_2`? How does the graph gain access to this schema if it was not passed in the `StateGraph` initialization? We can do this because _nodes can also declare additional state channels_ as long as the state schema definition exists. In this case, the `PrivateState` schema is defined, so we can add `bar` as a new state channel in the graph and write to it.

### Reducers¶

Reducers are key to understanding how updates from nodes are applied to the `State`. Each key in the `State` has its own independent reducer function. If no reducer function is explicitly specified then it is assumed that all updates to that key should override it. There are a few different types of reducers, starting with the default type of reducer:

#### Default Reducer¶

These two examples show how to use the default reducer:

**Example A:**

```
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]

```

In this example, no reducer functions are specified for any key. Let's assume the input to the graph is `{"foo": 1, "bar": ["hi"]}`. Let's then assume the first `Node` returns `{"foo": 2}`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{"foo": 2, "bar": ["hi"]}`. If the second node returns `{"bar": ["bye"]}` then the `State` would then be `{"foo": 2, "bar": ["bye"]}`

**Example B:**

```
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]

```

In this example, we've used the `Annotated` type to specify a reducer function (`operator.add`) for the second key (`bar`). Note that the first key remains unchanged. Let's assume the input to the graph is `{"foo": 1, "bar": ["hi"]}`. Let's then assume the first `Node` returns `{"foo": 2}`. This is treated as an update to the state. Notice that the `Node` does not need to return the whole `State` schema - just an update. After applying this update, the `State` would then be `{"foo": 2, "bar": ["hi"]}`. If the second node returns `{"bar": ["bye"]}` then the `State` would then be `{"foo": 2, "bar": ["hi", "bye"]}`. Notice here that the `bar` key is updated by adding the two lists together.

### Working with Messages in Graph State¶

#### Why use messages?¶

Most modern LLM providers have a chat model interface that accepts a list of messages as input. LangChain's `ChatModel` in particular accepts a list of `Message` objects as inputs. These messages come in a variety of forms such as `HumanMessage` (user input) or `AIMessage` (LLM response). To read more about what message objects are, please refer to this conceptual guide.

#### Using Messages in your Graph¶

In many cases, it is helpful to store prior conversation history as a list of messages in your graph state. To do so, we can add a key (channel) to the graph state that stores a list of `Message` objects and annotate it with a reducer function (see `messages` key in the example below). The reducer function is vital to telling the graph how to update the list of `Message` objects in the state with each state update (for example, when a node sends an update). If you don't specify a reducer, every state update will overwrite the list of messages with the most recently provided value. If you wanted to simply append messages to the existing list, you could use `operator.add` as a reducer.

However, you might also want to manually update messages in your graph state (e.g. human-in-the-loop). If you were to use `operator.add`, the manual state updates you send to the graph would be appended to the existing list of messages, instead of updating existing messages. To avoid that, you need a reducer that can keep track of message IDs and overwrite existing messages, if updated. To achieve this, you can use the prebuilt `add_messages` function. For brand new messages, it will simply append to existing list, but it will also handle the updates for existing messages correctly.

#### Serialization¶

In addition to keeping track of message IDs, the `add_messages` function will also try to deserialize messages into LangChain `Message` objects whenever a state update is received on the `messages` channel. See more information on LangChain serialization/deserialization here. This allows sending graph inputs / state updates in the following format:

```
# this is supported
{"messages": [HumanMessage(content="message")]}

# and this is also supported
{"messages": [{"type": "human", "content": "message"}]}

```

Since the state updates are always deserialized into LangChain `Messages` when using `add_messages`, you should use dot notation to access message attributes, like `state["messages"][-1].content`. Below is an example of a graph that uses `add_messages` as it's reducer function.

```
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

```

#### MessagesState¶

Since having a list of messages in your state is so common, there exists a prebuilt state called `MessagesState` which makes it easy to use messages. `MessagesState` is defined with a single `messages` key which is a list of `AnyMessage` objects and uses the `add_messages` reducer. Typically, there is more state to track than just messages, so we see people subclass this state and add more fields, like:

```
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]

```

## Nodes¶

In LangGraph, nodes are typically python functions (sync or `async`) where the **first** positional argument is the state, and (optionally), the **second** positional argument is a "config", containing optional configurable parameters (such as a `thread_id`).

Similar to `NetworkX`, you add these nodes to a graph using the add_node method:

```
from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph

builder = StateGraph(dict)


def my_node(state: dict, config: RunnableConfig):
    print("In node: ", config["configurable"]["user_id"])
    return {"results": f"Hello, {state['input']}!"}


# The second argument is optional
def my_other_node(state: dict):
    return state


builder.add_node("my_node", my_node)
builder.add_node("other_node", my_other_node)
...

```

Behind the scenes, functions are converted to RunnableLambda's, which add batch and async support to your function, along with native tracing and debugging.

If you add a node to graph without specifying a name, it will be given a default name equivalent to the function name.

```
builder.add_node(my_node)
# You can then create edges to/from this node by referencing it as `"my_node"`

```

### `START` Node¶

The `START` Node is a special node that represents the node sends user input to the graph. The main purpose for referencing this node is to determine which nodes should be called first.

```
from langgraph.graph import START

graph.add_edge(START, "node_a")

```

### `END` Node¶

The `END` Node is a special node that represents a terminal node. This node is referenced when you want to denote which edges have no actions after they are done.

```
from langgraph.graph import END

graph.add_edge("node_a", END)

```

## Edges¶

Edges define how the logic is routed and how the graph decides to stop. This is a big part of how your agents work and how different nodes communicate with each other. There are a few key types of edges:

  * Normal Edges: Go directly from one node to the next.
  * Conditional Edges: Call a function to determine which node(s) to go to next.
  * Entry Point: Which node to call first when user input arrives.
  * Conditional Entry Point: Call a function to determine which node(s) to call first when user input arrives.

A node can have MULTIPLE outgoing edges. If a node has multiple out-going edges, **all** of those destination nodes will be executed in parallel as a part of the next superstep.

### Normal Edges¶

If you **always** want to go from node A to node B, you can use the add_edge method directly.

```
graph.add_edge("node_a", "node_b")

```

### Conditional Edges¶

If you want to **optionally** route to 1 or more edges (or optionally terminate), you can use the add_conditional_edges method. This method accepts the name of a node and a "routing function" to call after that node is executed:

```
graph.add_conditional_edges("node_a", routing_function)

```

Similar to nodes, the `routing_function` accept the current `state` of the graph and return a value.

By default, the return value `routing_function` is used as the name of the node (or a list of nodes) to send the state to next. All those nodes will be run in parallel as a part of the next superstep.

You can optionally provide a dictionary that maps the `routing_function`'s output to the name of the next node.

```
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})

```

Tip

Use `Command` instead of conditional edges if you want to combine state updates and routing in a single function.

### Entry Point¶

The entry point is the first node(s) that are run when the graph starts. You can use the `add_edge` method from the virtual `START` node to the first node to execute to specify where to enter the graph.

```
from langgraph.graph import START

graph.add_edge(START, "node_a")

```

### Conditional Entry Point¶

A conditional entry point lets you start at different nodes depending on custom logic. You can use `add_conditional_edges` from the virtual `START` node to accomplish this.

```
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)

```

You can optionally provide a dictionary that maps the `routing_function`'s output to the name of the next node.

```
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})

```

## `Send`¶

By default, `Nodes` and `Edges` are defined ahead of time and operate on the same shared state. However, there can be cases where the exact edges are not known ahead of time and/or you may want different versions of `State` to exist at the same time. A common of example of this is with `map-reduce` design patterns. In this design pattern, a first node may generate a list of objects, and you may want to apply some other node to all those objects. The number of objects may be unknown ahead of time (meaning the number of edges may not be known) and the input `State` to the downstream `Node` should be different (one for each generated object).

To support this design pattern, LangGraph supports returning `Send` objects from conditional edges. `Send` takes two arguments: first is the name of the node, and second is the state to pass to that node.

```
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)

```

## `Command`¶

It can be useful to combine control flow (edges) and state updates (nodes). For example, you might want to BOTH perform state updates AND decide which node to go to next in the SAME node. LangGraph provides a way to do so by returning a `Command` object from node functions:

```
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )

```

With `Command` you can also achieve dynamic control flow behavior (identical to conditional edges):

```
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")

```

Important

When returning `Command` in your node functions, you must add return type annotations with the list of node names the node is routing to, e.g. `Command[Literal["my_other_node"]]`. This is necessary for the graph rendering and tells LangGraph that `my_node` can navigate to `my_other_node`.

Check out this how-to guide for an end-to-end example of how to use `Command`.

### Using inside tools¶

A common use case is updating graph state from inside a tool. For example, in a customer support application you might want to look up customer information based on their account number or ID in the beginning of the conversation. To update the graph state from the tool, you can return `Command(update={"my_custom_key": "foo", "messages": [...]})` from the tool:

```
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # update the state keys
            "user_info": user_info,
            # update the message history
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )

```

Important

You MUST include `messages` (or any state key used for the message history) in `Command.update` when returning `Command` from a tool and the list of messages in `messages` MUST contain a `ToolMessage`. This is necessary for the resulting message history to be valid (LLM providers require AI messages with tool calls to be followed by the tool result messages).

If you are using tools that update state via `Command`, we recommend using prebuilt `ToolNode` which automatically handles tools returning `Command` objects and propagates them to the graph state. If you're writing a custom node that calls tools, you would need to manually propagate `Command` objects returned by the tools as the update from node.

## Persistence¶

LangGraph provides built-in persistence for your agent's state using checkpointers. Checkpointers save snapshots of the graph state at every superstep, allowing resumption at any time. This enables features like human-in-the-loop interactions, memory management, and fault-tolerance. You can even directly manipulate a graph's state after its execution using the appropriate `get` and `update` methods. For more details, see the persistence conceptual guide.

## Threads¶

Threads in LangGraph represent individual sessions or conversations between your graph and a user. When using checkpointing, turns in a single conversation (and even steps within a single graph execution) are organized by a unique thread ID.

## Storage¶

LangGraph provides built-in document storage through the BaseStore interface. Unlike checkpointers, which save state by thread ID, stores use custom namespaces for organizing data. This enables cross-thread persistence, allowing agents to maintain long-term memories, learn from past interactions, and accumulate knowledge over time. Common use cases include storing user profiles, building knowledge bases, and managing global preferences across all threads.

## Graph Migrations¶

LangGraph can easily handle migrations of graph definitions (nodes, edges, and state) even when using a checkpointer to track state.

  * For threads at the end of the graph (i.e. not interrupted) you can change the entire topology of the graph (i.e. all nodes and edges, remove, add, rename, etc)
  * For threads currently interrupted, we support all topology changes other than renaming / removing nodes (as that thread could now be about to enter a node that no longer exists) -- if this is a blocker please reach out and we can prioritize a solution.
  * For modifying state, we have full backwards and forwards compatibility for adding and removing keys
  * State keys that are renamed lose their saved state in existing threads
  * State keys whose types change in incompatible ways could currently cause issues in threads with state from before the change -- if this is a blocker please reach out and we can prioritize a solution.

## Configuration¶

When creating a graph, you can also mark that certain parts of the graph are configurable. This is commonly done to enable easily switching between models or system prompts. This allows you to create a single "cognitive architecture" (the graph) but have multiple different instance of it.

You can optionally specify a `config_schema` when creating a graph.

```
class ConfigSchema(TypedDict):
    llm: str

graph = StateGraph(State, config_schema=ConfigSchema)

```

You can then pass this configuration into the graph using the `configurable` config field.

```
config = {"configurable": {"llm": "anthropic"}}

graph.invoke(inputs, config=config)

```

You can then access and use this configuration inside a node:

```
def node_a(state, config):
    llm_type = config.get("configurable", {}).get("llm", "openai")
    llm = get_llm(llm_type)
    ...

```

See this guide for a full breakdown on configuration.

### Recursion Limit¶

The recursion limit sets the maximum number of super-steps the graph can execute during a single execution. Once the limit is reached, LangGraph will raise `GraphRecursionError`. By default this value is set to 25 steps. The recursion limit can be set on any graph at runtime, and is passed to `.invoke`/`.stream` via the config dictionary. Importantly, `recursion_limit` is a standalone `config` key and should not be passed inside the `configurable` key as all other user-defined configuration. See the example below:

```
graph.invoke(inputs, config={"recursion_limit": 5, "configurable":{"llm": "anthropic"}})

```

Read this how-to to learn more about how the recursion limit works.

## Breakpoints¶

It can often be useful to set breakpoints before or after certain nodes execute. This can be used to wait for human approval before continuing. These can be set when you "compile" a graph. You can set breakpoints either _before_ a node executes (using `interrupt_before`) or after a node executes (using `interrupt_after`.)

You **MUST** use a checkpointer when using breakpoints. This is because your graph needs to be able to resume execution.

In order to resume execution, you can just invoke your graph with `None` as the input.

```
# Initial run of graph
graph.invoke(inputs, config=config)

# Let's assume it hit a breakpoint somewhere, you can then resume by passing in None
graph.invoke(None, config=config)

```

See this guide for a full walkthrough of how to add breakpoints.

### Dynamic Breakpoints¶

It may be helpful to **dynamically** interrupt the graph from inside a given node based on some condition. In `LangGraph` you can do so by using `NodeInterrupt` \-- a special exception that can be raised from inside a node.

```
def my_node(state: State) -> State:
    if len(state['input']) > 5:
        raise NodeInterrupt(f"Received input that is longer than 5 characters: {state['input']}")

    return state

```

## Subgraphs¶

A subgraph is a graph that is used as a node in another graph. This is nothing more than the age-old concept of encapsulation, applied to LangGraph. Some reasons for using subgraphs are:

  * building multi-agent systems

  * when you want to reuse a set of nodes in multiple graphs, which maybe share some state, you can define them once in a subgraph and then use them in multiple parent graphs

  * when you want different teams to work on different parts of the graph independently, you can define each part as a subgraph, and as long as the subgraph interface (the input and output schemas) is respected, the parent graph can be built without knowing any details of the subgraph

There are two ways to add subgraphs to a parent graph:

  * add a node with the compiled subgraph: this is useful when the parent graph and the subgraph share state keys and you don't need to transform state on the way in or out

```
builder.add_node("subgraph", subgraph_builder.compile())

```

  * add a node with a function that invokes the subgraph: this is useful when the parent graph and the subgraph have different state schemas and you need to transform state before or after calling the subgraph

```
subgraph = subgraph_builder.compile()

def call_subgraph(state: State):
    return subgraph.invoke({"subgraph_key": state["parent_key"]})

builder.add_node("subgraph", call_subgraph)

```

Let's take a look at examples for each.

### As a compiled graph¶

The simplest way to create subgraph nodes is by using a compiled subgraph directly. When doing so, it is **important** that the parent graph and the subgraph state schemas share at least one key which they can use to communicate. If your graph and subgraph do not share any keys, you should use write a function invoking the subgraph instead.

Note

If you pass extra keys to the subgraph node (i.e., in addition to the shared keys), they will be ignored by the subgraph node. Similarly, if you return extra keys from the subgraph, they will be ignored by the parent graph.

```
from langgraph.graph import START, StateGraph
from typing import TypedDict

class State(TypedDict):
    foo: str

class SubgraphState(TypedDict):
    foo: str  # note that this key is shared with the parent graph state
    bar: str

# Define subgraph
def subgraph_node(state: SubgraphState):
    # note that this subgraph node can communicate with the parent graph via the shared "foo" key
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node)
...
subgraph = subgraph_builder.compile()

# Define parent graph
builder = StateGraph(State)
builder.add_node("subgraph", subgraph)
...
graph = builder.compile()

```

### As a function¶

You might want to define a subgraph with a completely different schema. In this case, you can create a node function that invokes the subgraph. This function will need to transform the input (parent) state to the subgraph state before invoking the subgraph, and transform the results back to the parent state before returning the state update from the node.

```
class State(TypedDict):
    foo: str

class SubgraphState(TypedDict):
    # note that none of these keys are shared with the parent graph state
    bar: str
    baz: str

# Define subgraph
def subgraph_node(state: SubgraphState):
    return {"bar": state["bar"] + "baz"}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node)
...
subgraph = subgraph_builder.compile()

# Define parent graph
def node(state: State):
    # transform the state to the subgraph state
    response = subgraph.invoke({"bar": state["foo"]})
    # transform response back to the parent state
    return {"foo": response["bar"]}

builder = StateGraph(State)
# note that we are using `node` function instead of a compiled subgraph
builder.add_node(node)
...
graph = builder.compile()

```

## Visualization¶

It's often nice to be able to visualize graphs, especially as they get more complex. LangGraph comes with several built-in ways to visualize graphs. See this how-to guide for more info.

## Streaming¶

LangGraph is built with first class support for streaming, including streaming updates from graph nodes during the execution, streaming tokens from LLM calls and more. See this conceptual guide for more information.

## Comments

Previous

Why LangGraph?

Next

Agent architectures

Made with  Material for MkDocs Insiders