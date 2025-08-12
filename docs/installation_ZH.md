# 1、安装

## 1.1 环境安装

```markdown
# Python 虚拟环境安装
# 配置国内镜像源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 安装包管理
pip  install  uv   --pip-args="--index-url https://pypi.tuna.tsinghua.edu.cn/simple"  # 国内镜像加速


# 创建项目目录
mkdir image_project

# 初始化项目
cd  image_project

# pyproject.toml 文件末尾中添加
[tool.uv]
index-url = "https://pypi.tuna.tsinghua.edu.cn/simple"
# 安装依赖
uv add  langgraph

# 安装模型接口
uv add  langchain-deepseek  langchain-anthropic langchain-openai langchain-ollama

# 激活虚拟环境
source ./.venv/bin/activate

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 安装依赖
pip install -U langgraph

```

## 1.2   agent编排

vi .env 文件

```
# OpenAI 配置
OPENAI_MODEL_NAME=gpt-4.1-mini-2025-04-14
OPENAI_API_KEY=xxxx
OPENAI_BASE_URL=xxx

# DeepSeek 
DEEPSEEK_MODEL_NAME=deepseek-chat
DEEPSEEK_API_KEY=xxxxx
DEEPSEEK_BASE_URL=https://api.deepseek.com  # 如果是自定义代理，替换此处

```

vi  basic_config.py

```
from pydantic import Field, BaseModel
from pydantic_settings import BaseSettings,SettingsConfigDict
from dotenv import load_dotenv
from dotenv import load_dotenv
from pathlib import Path
import os

# .env 文件位于你运行脚本的同一目录
load_dotenv(dotenv_path=Path(__file__).parent / ".env")

class OpenAIConfig(BaseSettings):
    model_name: str = Field(..., validation_alias="OPENAI_MODEL_NAME", description="OpenAI 模型名称")
    api_key: str = Field(..., validation_alias="OPENAI_API_KEY", description="OpenAI API 密钥")
    base_url: str = Field(default="https://api.openai.com/v1", validation_alias="OPENAI_BASE_URL", description="OpenAI 接口地址")
    model_config = SettingsConfigDict(env_file_encoding='utf-8')


class DeepSeekConfig(BaseSettings):
    model_name: str = Field(..., validation_alias="DEEPSEEK_MODEL_NAME", description="DeepSeek 模型名称")
    api_key: str = Field(..., validation_alias="DEEPSEEK_API_KEY", description="DeepSeek API 密钥")
    base_url: str = Field(default="https://api.deepseek.com", validation_alias="DEEPSEEK_BASE_URL", description="DeepSeek 接口地址")
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
    """模拟一个搜索工具"""

    if "上海" in query.lower() or"Shanghai"in query.lower():
        return "现在30度，有雾."

    return "现在是35度，阳光明媚。"

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

# 将工具函数放入工具列表
tools=[search,human_assistance] 

# 创建工具节点
tool_node = ToolNode(tools=tools)

llm_with_tools = llm.bind_tools(tools)


# 定义函数，决定是否继续执行
def should_continue(state: MessagesState) ->Literal["tools", END]:
    messages = state['messages']
    last_message = messages[-1]
    # 如果LLM调用了工具，则转到“tools”节点
    if last_message.tool_calls:
        return"tools"
    # 否则，停止（回复用户）
    return END


# 定义调用模型的函数
def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}


# 2.用状态初始化图，定义一个新的状态图
graph_builder=StateGraph(State)

# 3.定义图节点，定义我们将循环的两个节点
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", tool_node)


# 4.定义入口点和图边、设置入口点为“chatbot”、这意味着这是第一个被调用的节点
graph_builder.set_entry_point("chatbot")



# 添定义条件边将控制流从一个节点路由到下一个节点。
graph_builder.add_conditional_edges(
   # 首先，定义起始节点。我们使用`chatbot`，这意味着这些边是在调用`agent`节点后采取的。
    "chatbot",
    # 接下来，传递决定下一个调用节点的函数。
    should_continue,
)


# 添加从`tools`到`chatbot`的普通边。这意味着在调用`tools`后，接下来调用`chatbot`节点。
graph_builder.add_edge("tools", "chatbot")

# 5、初始化内存以在图运行之间持久化状态
memory = MemorySaver()

# 6、编译图--编译成一个LangChain可运行对象--我们（可选地）在编译图时传递内存
graph = graph_builder.compile(checkpointer=memory)


# 7、执行图
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

