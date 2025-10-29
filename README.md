# A2A Demo

![示意图](./pics/Demo示意图.png)

- 用户在浏览器端发出指令后，前端会将该请求传给 Host Agent，由它负责解析用户意图、拆解具体子任务，并且并行触发多个 Remote Agent；
- 每个 Remote Agent 通过 A2A Client 将子任务封装为标准的 JSON-RPC 请求，发送给远端对应的 A2A Server，再由后者调用各自擅长的智能体模块（如 LangGraph Agent 负责外汇兑换、Google ADK Agent 负责报销收据、Crew AI Agent 负责根据文字内容来生成图片等）执行并返回结果；
- 最后，Host Agent 汇总并格式化各路反馈，一并呈现给用户，实现多智能体的分工协作与能力互补。

---

## Prerequisites
- Python 3.13 or higher
- [UV](https://docs.astral.sh/uv/)

在根目录创建 .env 文件，写入GOOGLE_API_KEY:
```
GOOGLE_API_KEY=<your_google_api_key>
```
Google_API_Key[申请地址](https://cloud.google.com/docs/authentication/api-keys?hl=zh-cn)

## Running the Samples

Run one (or more) [agent](/samples/python/agents/README.md) A2A server and one of the [host applications](/samples/python/hosts/README.md). 

The following example will run the langgraph agent with the python CLI host:

1. Navigate to the agent directory:
    ```bash
    cd samples/python/agents/langgraph
    ```
2. Run an agent:
    ```bash
    uv run .
    ```
3. In another terminal, navigate to the CLI directory:
    ```bash
    cd samples/python/hosts/cli
    ```
4. Run the example client
    ```
    uv run .
    ```
---

**NOTE:** 
This is sample code and not production-quality libraries.

---

## issue
https://github.com/huangjia2019/a2a-in-action/issues/1

---

## A2A整体架构
从整体架构上看，一个 A2A 系统由以下主要组件构成：
- Agent Card: 代理的身份标识和能力声明。
- Skill: 代理可执行的具体能力。
- Task Manager: 处理任务流和状态管理。
- Server: 提供 HTTP 接口让其他代理 / 客户端访问

在这些关键组件实现的基础之上，还需要为 Demo 应用创建 UI，并注册服务，还要通过一系列的通信机制让外部 Agent 和 Host Agent（本地 Agent）能够相互对话，进行能力的发现。

---

## Sample Code
目录结构：
- agents目录：包含一系列 A2A Agents 示例。
- Demo目录：是要演示的协议实战示例。
- Common目录：Common code that all sample agents and apps use to speak A2A over HTTP. 
- Hosts目录：Host applications that use the A2AClient. Includes a CLI which shows simple task completion with a single agent, a mesop web application that can speak to multiple agents, and an orchestrator agent that delegates tasks to one of multiple remote A2A agents.

### UI层入口与服务注
- 在文件夹 demo/ui/pages.py 中，就这个应用的前端页面;
- 文件 demo/ui/main.py 中通过  FastAPI  启动后端服务，并用  Mesop  框架组织前端页面。

### Host Agent 服务实现
前端服务需要首先和本地 Agent 建立连接，才能实现和用户的交互对话。文件 demo/ui/service/server/server.py 中的 ConversationServer  类是  UI  与  Host Agent  之间的桥梁，负责路由注册和请求分发。初始化时会根据环境变量选择  ADKHostManager（真实多智能体调度）或 InMemoryFakeAgentManager（假数据）。
- async def _send_message(self, request: Request)，它接收前端消息，调用  self.manager.process_message(message)  进行处理（异步线程），并返回消息 ID。
- async def _register_agent(self, request: Request)，支持动态注册远程 Agent，调用  self.manager.register_agent(url)。
- async def _list_agents(self)，返回当前已注册的所有 Agent 信息。

### Host Agent  调度与 A2A 协议实现
在文件 demo/ui/service/server/adk_host_manager.py 中，ADKHostManager  继承自  ApplicationManager，是真正的“智能体大脑”。
- self._host_agent = HostAgent([], self.task_callback)，这里的  HostAgent  是多智能体调度的核心，负责多智能体的注册、能力发现、任务分发、回调等，是  Host  侧的“大脑”。这个类的定义位于 hosts/multiagent/host_agent.py  文件。
- async def process_message(self, message: Message)，处理用户消息，维护会话、消息、事件等，并通过  self._host_runner.run_async(…)  触发智能体推理和任务流转。
- def register_agent(self, url)，支持通过 URL 动态注册远程 Agent，Agent 信息会被加入  _agents  列表，供 Host 调度。
- @property def agents(self)，返回所有已注册的 AgentCard（能力描述卡片），用于能力发现和展示。

HostAgent  通过  remote_agent_connection.py  维护与每个远程 Agent 的连接和能力卡片（AgentCard）—— 这也是 A2A 协议的核心交互机制。

---

### Remote Agents 和 Agent Card
在 agent 目录中，有一系列可以配置到 Demo 应用中的外部（Remote）Agents，每个子目录就是一个外部 Agent 的实现。
- agents/langgraph/agent.py 中的 CurrencyAgent，用  LangGraph 框架和 Google Gemini API 实现了货币兑换的智能体逻辑，支持流式和同步调用。
- agents/langgraph/main.py 中通过  A2AServer（A2A 协议服务端实现）将  CurrencyAgent  以  HTTP  服务形式暴露出来，并注册了自己的  AgentCard，即能力卡片。


Agent Card 是 A2A 协议的关键内核概念之一，它负责定义代理的元数据（名称、描述、URL、版本等），声明支持的输入 / 输出模态（如文本、图像等）同时列出代理提供的技能清单（skill），其目的是让你的代理能够被发现，并让其他系统知道如何与它交互。这是实现“代理之间互相通信”（Agent-to-Agent）的基础。
```py
agent_card = AgentCard(
    name='Currency Agent',
    description='Helps with exchange rates for currencies',
    url=f'http://{host}:{port}/',
    version='1.0.0',
    defaultInputModes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
    defaultOutputModes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
    capabilities=capabilities,
    skills=[skill],
)
```

A2AServer 则是 A2A 的协议服务端实现，其中的 Task Manager 负责任务生命周期管理，状态追踪和更新（WORKING、COMPLETED、ERROR 等），处理同步 / 异步请求（on_send_task  和  on_send_task_subscribe），生成适当的响应格式以及错误处理和恢复。
```py
server = A2AServer(
      agent_card=agent_card,
      task_manager=AgentTaskManager(
          agent=CurrencyAgent(),
          notification_sender_auth=notification_sender_auth,
      ),
      host=host,
      port=port,
  )
```

---

### A2A Client  端实现
文件 demo/ui/service/client/client.py 中的 ConversationClient  类封装了与远程 Agent 的 HTTP 通信，所有请求都以  JSON-RPC  格式发送。
此处的关键方法如下：
- async def send_message(self, payload: SendMessageRequest)，通过 HTTP POST 将消息发送到远程 Agent 的  /message/send  接口。
- async def register_agent(self, payload: RegisterAgentRequest)，远程注册 Agent。