# 1、Install

## 1.1 Environmental installation

```markdown
# Python Virtual Environment Installation
# Configure domestic image source
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# Installation package management
pip  install  uv   --pip-args="--index-url https://pypi.tuna.tsinghua.edu.cn/simple"  # 国内镜像加速


# Create project directory
mkdir image_project

# Initialize Project
cd  image_project

#  Add at the end of the file pyproject.toml 
[tool.uv]
index-url = "https://pypi.tuna.tsinghua.edu.cn/simple"

# Install dependencies
uv add  langgraph

# Install model interface
uv add  langchain-deepseek  langchain-anthropic langchain-openai langchain-ollama

# Activate virtual environment
source ./.venv/bin/activate

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# Install dependencies
pip install -U langgraph

```

## 1.2   Agent orchestration

vi .env file

```
# OpenAI config
OPENAI_MODEL_NAME=gpt-4.1-mini-2025-04-14
OPENAI_API_KEY=xxxx
OPENAI_BASE_URL=xxx

# DeepSeek 
DEEPSEEK_MODEL_NAME=deepseek-chat
DEEPSEEK_API_KEY=xxxxx
DEEPSEEK_BASE_URL=https://api.deepseek.com  # f it is a custom proxy, replace here

```

vi  basic_config.py

```
from pydantic import Field, BaseModel
from pydantic_settings import BaseSettings,SettingsConfigDict
from dotenv import load_dotenv
from dotenv import load_dotenv
from pathlib import Path
import os

# .env The file is located in the same directory where you run the script
load_dotenv(dotenv_path=Path(__file__).parent / ".env")

class OpenAIConfig(BaseSettings):
    model_name: str = Field(..., validation_alias="OPENAI_MODEL_NAME", description="OpenAI model name")
    api_key: str = Field(..., validation_alias="OPENAI_API_KEY", description="OpenAI API secret key")
    base_url: str = Field(default="https://api.openai.com/v1", validation_alias="OPENAI_BASE_URL", description="OpenAI url")
    model_config = SettingsConfigDict(env_file_encoding='utf-8')


class DeepSeekConfig(BaseSettings):
    model_name: str = Field(..., validation_alias="DEEPSEEK_MODEL_NAME", description="DeepSeek model name")
    api_key: str = Field(..., validation_alias="DEEPSEEK_API_KEY", description="DeepSeek API 密钥")
    base_url: str = Field(default="https://api.deepseek.com", validation_alias="DEEPSEEK_BASE_URL", description="DeepSeek base url")
    model_config = SettingsConfigDict(env_file_encoding='utf-8')


if __name__ == "__main__":
    try:
        openai_config = OpenAIConfig()
        print(f"OpenAI Model Name: {openai_config.model_name}")
        print(f"OpenAI API Key: {openai_config.api_key}")

        deepseek_config = DeepSeekConfig()
        print(f"DeepSeek Model Name: {deepseek_config.model_name}")
        print(f"DeepSeek API Key: {deepseek_config.api_key}")
    except Exception as e:

```



vi agent-v1.py

```markdown
from langchain_deepseek import ChatDeepSeek
from typing import Annotated,Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph,START,END,MessagesState
from langgraph.graph.message import add_messages
from langgraph.prebuilt import create_react_agent
from basic_config import DeepSeekConfig,OpenAIConfig
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import ToolNode
from langgraph.types import Command, interrupt
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage,SystemMessage

openai_config = OpenAIConfig()
deepseek_config = DeepSeekConfig()

# print(openai_config.base_url)
llm=ChatDeepSeek(
    base_url=deepseek_config.base_url,
    model=deepseek_config.model_name,
    api_key=deepseek_config.api_key,
    temperature=0
)


class State(TypedDict):
    messages: Annotated[list, add_messages]



@tool
def search(query: str):
'''
   Simulate a search tool
If "Shanghai" in query. lower() or "Shanghai" in query. lower():
It's currently 30 degrees and foggy "
The current temperature is 35 degrees and the sun is shining brightly. "
'''

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

# Put the utility function into the tool list
tools=[search,human_assistance] 

# Create tool node
tool_node = ToolNode(tools=tools)
llm_with_tools = llm.bind_tools(tools)

# Define a function to determine whether to continue execution
def should_continue(state: MessagesState) ->Literal["tools", END]:
    messages = state['messages']
    last_message = messages[-1]
    # If LLM calls a tool, go to the 'tools' node
    if last_message.tool_calls:
        return"tools"
    return END


# Define the functions that call the model
def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}


# 3. Define graph nodes and define the two nodes we will loop through
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", tool_node)

# 4. Define the entry point and graph edge, set the entry point as "chatbot", which means this is the first node to be called
graph_builder.set_entry_point("chatbot")
#Adding conditional edges to route control flow from one node to the next.
graph_builder.add_conditional_edges(

# Firstly, define the starting node. We use 'chatbot', which means that these edges are taken after calling the 'agent' node.
"chatbot",
#Next, pass the function that determines the next calling node.
should_continue,
)
# Add regular edges from tools to chatbot. This means that after calling 'tools', the next step is to call the' chatbot 'node.
graph_builder.add_edge("tools", "chatbot")

# 5. Initialize memory to persist state between graph runs
memory = MemorySaver()

# 6. Compile Graph - Compile into a LangChain runnable object - We (optionally) pass memory when compiling the graph
graph = graph_builder.compile(checkpointer=memory)

# 7. Execution diagram
def exe_graph(user_input: str):
    final_state = graph.invoke(
    {"messages": [HumanMessage(content=user_input)]},
    config={"configurable": {"thread_id": 42}})
    # 从 final_state 中获取最后一条消息的内容
    result = final_state["messages"][-1].content
    print(result)

    final_state = graph.invoke(
    {"messages": [HumanMessage(content="我问的那个城市?")]},
    config={"configurable": {"thread_id": 42}})
    result = final_state["messages"][-1].content
    print(result)


def stream_graph_updates(user_input: str):
    events=graph.stream(
        input={"messages": [{"role": "user", "content": user_input}]},
        config={"configurable": {"thread_id": "1"}},
        stream_mode="values",
    )
    for event in events:
        if "messages" in event:
            event["messages"][-1].pretty_print()


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
```

