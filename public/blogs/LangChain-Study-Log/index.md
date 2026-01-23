# **1.简易对话链**
StrOutputParser 是 LangChain 框架中最基础且使用频率最高的输出解析器（Output Parser）组件。 ‌,它的核心功能是将大型语言模型（LLM）生成的原始文本回复，‌原样返回‌，不进行任何结构化转换或内容修改。 ‌

**主要用途与场景:**

1. 提取纯文本内容‌：当 LLM 的响应中包含工具调用（tool calling）的复杂结构时，StrOutputParser 可以自动提取出其中的纯文本部分，便于后续处理。 ‌

2. 简化流程‌：在不需要将输出转换为字典、JSON 或 Pydantic 对象等结构化数据的场景下，使用 StrOutputParser 可以直接获取模型的文本输出。 ‌

3. 链式调用集成‌：作为 LangChain 表达式语言（LCEL）的一部分，它可以无缝地与其他组件通过 | 操作符组合使用。 ‌
```
#必要库
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
```
```
#1、定义模型
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

#2、构建链:prompt ->llm->解析文本
qa_chain = (
    ChatPromptTemplate.from_template("用户问题:{question}")
    | llm
    | StrOutputParser()
)

#3、执行问答
if __name__ == "__main__":
    result = qa_chain.invoke({"question":"中国的首都是哪里?"})
    print("AI答复:",result)
```
```
AI答复: 中国的首都是北京。
```

# **2.结构化输出**
**语法含义拆解**
1. class AnalysisResult(BaseModel):
BaseModel: 这是 Pydantic 的基类。继承它之后，AnalysisResult 就拥有了自动校验、序列化（如 .dict(), .json()）等高级功能。

2. sentiment: Literal['积极','中性','消极']
类型注解: 声明 sentiment 字段。

Literal（字面量约束）: 这是 Python typing 模块的功能。它强制要求该字段的值必须且只能是 '积极'、'中性' 或 '消极' 中的一个。如果传入“开心”或“愤怒”，代码在校验阶段会报错。

3. Field(..., description="...")
Field 函数: 用于为字段添加额外的元数据（Metadata）。

... (Ellipsis): 在这里表示该字段是必填的。如果没有提供该字段，程序会抛出 ValidationError。

description: 描述性文字。这在生成 API 文档（如 FastAPI）或者给 LLM 提供 Prompt（提示词）时非常有用，告诉程序或 AI 这个字段的具体含义。

4. topic: str 与 summary: str
定义了两个字符串类型的字段，分别对应“主要主题”和“简要总结”。

5. 提示词（Prompt）是控制 AI 行为的核心。

{text}: 这是一个占位符，稍后在调用时会被真实的待分析文本替换。

{{ ... }} (双大括号):

语法含义: 在 Python 的字符串格式化和 LangChain 模板中，单大括号 {} 用于变量注入。

作用: 为了在 Prompt 中保留 JSON 原始的 { 符号而不被当作变量解析，必须使用双大括号进行转义。

少样本提示 (Few-Shot): 通过提供“示例输出格式”，给模型一个参考标准，提高解析成功率。

6. LangChain 的核心语法——LCEL (LangChain Expression Language)。

| (管道操作符): 类似于 Linux 的管道。它将上一个环节的输出（格式化后的 Prompt）作为下一个环节（LLM）的输入。

with_structured_output(AnalysisResult):

关键功能: 这是 LangChain 提供的极简接口。它会自动处理复杂的逻辑：

将 AnalysisResult 模型转换成 JSON Schema。

将该 Schema 传递给模型（通常通过 Tool Calling 或 JSON Mode）。

自动解析模型返回的字符串，并将其直接转换成 AnalysisResult 对象

7. 执行与异常处理

.invoke({"text": "..."}): 启动链式调用，传入 text 变量的值。

result: AnalysisResult: 类型提示，明确 invoke 返回的是一个 AnalysisResult 类的实例，而不是普通的字符串或字典。

.model_dump():

语法含义: 这是 Pydantic v2 的方法（旧版为 .dict()）。

功能: 将 Pydantic 对象转换为标准的 Python 字典，便于后续打印或保存。

try...except 结构: 针对 AI 输出不稳定的防御性编程。如果 AI 返回了无法解析的错误 JSON，程序不会直接崩溃，而是会打印原始返回内容（raw_response.content）以便开发者排查问题。
```
#必要库
from typing import Literal
from pydantic import BaseModel,Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
```
```
# #1、定义输出Schema
# class AnalysisResult(BaseModel):
#     sentiment:Literal['积极','中性','消极'] = Field(...,description = "文本的情感极性")
#     topic:str = Field(...,description="主要主题")
#     summary:str = Field(...,description="简要总结")

# #2、定义模型
# llm = ChatOpenAI(
#     model = 'GLM-4.1V-Thinking-Flash',
#     base_url = "https://www.dmxapi.cn/v1",
#     api_key = "sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
#     temperature = 0
# )

# #3、构建LCEL链Prompt|LLM
# analysis_chain = (
#     ChatPromptTemplate.from_template("请分析一下文本，并提取关键信息：{text}")
#     | llm.with_structured_output(AnalysisResult)
# )

# #4、执行
# if __name__ =="__main__":
#     result:AnalysisResult = analysis_chain.invoke(
#         {"text":"今天股市大涨，我对未来的投资充满信心"}
#     )
#     print(result.dict())

# 1、定义输出Schema
class AnalysisResult(BaseModel):
    sentiment: Literal['积极','中性','消极'] = Field(..., description="文本的情感极性")
    topic: str = Field(..., description="主要主题")
    summary: str = Field(..., description="简要总结")

# 2、定义模型
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0,
    # 显式指定响应格式为JSON（部分模型需要）
    response_format={"type": "json_object"}
)

# 3、构建LCEL链（关键：强化Prompt，强制JSON格式+示例）
prompt_template = """
请严格按照以下要求分析文本并输出JSON格式结果，不要输出任何额外文字：
1. 情感极性（sentiment）只能是：积极/中性/消极
2. 主要主题（topic）：简要概括文本核心主题
3. 简要总结（summary）：10-30字总结文本内容
4. 输出必须是标准JSON，无需其他解释性文字

文本内容：{text}

示例输出格式：
{{
    "sentiment": "积极",
    "topic": "股市大涨对投资信心的影响",
    "summary": "今日股市大涨，用户对未来投资充满信心"
}}
"""

analysis_chain = (
    ChatPromptTemplate.from_template(prompt_template)
    | llm.with_structured_output(AnalysisResult)
)

# 4、执行（修复dict() → model_dump()，添加异常处理）
if __name__ == "__main__":
    try:
        result: AnalysisResult = analysis_chain.invoke(
            {"text": "今天股市大涨，我对未来的投资充满信心"}
        )
        print("解析成功：")
        print(result.model_dump())  
        # 如果需要美化输出，可结合json模块
        # import json
        # print(json.dumps(result.model_dump(), ensure_ascii=False, indent=2))
    except Exception as e:
        print(f"解析失败，错误信息：{e}")
        # 打印原始返回内容，排查格式问题
        raw_response = llm.invoke(
            ChatPromptTemplate.from_template(prompt_template).format(
                text="今天股市大涨，我对未来的投资充满信心"
            )
        )
        print("模型原始返回：", raw_response.content)
```
```
解析成功：
{'sentiment': '积极', 'topic': '股市上涨与投资信心', 'summary': '今日股市大涨，对未来投资充满信心'}
```
# **3.历史记忆与消息占位符**
在 LangChain 框架中，ChatPromptTemplate 的角色可以被理解为 “提示词工程的脚手架”。它不仅仅是一个简单的字符串格式化工具，更是连接“用户原始输入”与“模型结构化指令”之间的重要桥梁。

对话式模型（如 GLM-4, GPT-4）通常需要区分不同的角色。ChatPromptTemplate 允许你将提示词拆解为：

System Message: 设定 AI 的角色、行为准则（例如：“你是一个专业金融分析师”）。

Human Message: 用户的具体提问。

AI Message: 预设的 AI 回复（用于 Few-shot 示例）。
```
#必要库
from langchain_core.prompts import ChatPromptTemplate,MessagesPlaceholder
from langchain_core.messages import HumanMessage,AIMessage
from langchain_openai import ChatOpenAI
```
```
#1、定义Prompt包含历史消息占位符
prompt = ChatPromptTemplate.from_messages(
    [
        ("system","你是一个友好的中文助理"),
        MessagesPlaceholder("history"), #占位符历史消息注入此处
        ("human","{question}") #当前用户问题
    ]
)

#2、构造历史消息
history = [
    HumanMessage(content="1+1 = ?"),
    AIMessage(content="1+1 = 2")
]

#3、格式化prompt，把历史消息+新问题合并
prompt_value = prompt.format_messages(history = history,question = "我刚才问的问题是什么？")
print(prompt_value)

#4、定义大模型
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0,
)

#5、执行调用
response = llm.invoke(prompt_value)
print("AI答复:",response.content)
```
```
[SystemMessage(content='你是一个友好的中文助理', additional_kwargs={}, response_metadata={}), HumanMessage(content='1+1 = ?', additional_kwargs={}, response_metadata={}), AIMessage(content='1+1 = 2', additional_kwargs={}, response_metadata={}), HumanMessage(content='我刚才问的问题是什么？', additional_kwargs={}, response_metadata={})]
AI答复: 你刚才问的是“1+1 = ?”（或类似表达），当时助理回答了“1+1 = 2” 。如果是想确认具体问题，就是那个加法运算的提问~
```
# **4.工具调用(Tool Calling)**
在使用 LangChain 的 @tool 装饰器时，函数必须包含“文档字符串（docstring）”，或者在装饰器中显式提供 description 参数。这是因为 LangChain 需要将这个描述发送给大模型（LLM），让模型知道在什么情况下该调用这个工具。在这里的示例中就是@tool下面的 **"""计算两个整数之和。"""** 
```
#必要库
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langchain_core.messages import ToolMessage,AIMessage
```
```
# 定义一个工具：比如计算两位数之和
@tool
def add(a:int,b:int)->int:
    """计算两个整数之和。"""  # 必须添加这个描述。
    print("tool used...")
    return a+b

#模型绑定工具
llm = ChatOpenAI(
    model="GLM-4.5-Flash",  # 这里最好不要用thinking模型，因为thinking模型在输出前有一段推理过程，一些简单的工具函数可能在推理阶段就已经推出了答案，不会触发工具使用。
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)
llm_tools = llm.bind_tools([add]) # 如果有多个工具函数，每个函数都要按照规范写（@tool和"""解释文案"""），然后用列表的形式输入bind_tools方法

# 用户提问
user_input ="请帮我算一下 12 + 30等于多少，如果需要可以调用工具。"
# 第一次调用模型->可能触发工具
ai_msg = llm_tools.invoke(user_input)

# 如果模型触发了工具调用
if ai_msg.tool_calls:
    for call in ai_msg.tool_calls:
        if call["name"] == "add": # 执行工具
            result = add.invoke(call["args"])
            tool_msg = ToolMessage(content=str(result),name = call["name"],tool_call_id=call["id"])
            # 再次调用模型，让模型基于工具结果生成自然语言回答
            final_ai = llm_tools.invoke([("human",user_input),ai_msg,tool_msg])
            print("调用工具最终回答：",final_ai.content)
else:
    # 模型没有调用工具则直接回答
    print("直接回答：",ai_msg.content)
```
```
tool used...
调用工具最终回答： 
12 + 30 = 42
```
# 5.流式输出(Streaming)
这里通过在ChatOpenAI类中设置straming和callbacks参数实现AI流式回复效果，就和打字机一样，文本是一个一个快速弹出的。
```
#必要库
from langchain_core.callbacks import StreamingStdOutCallbackHandler
from langchain_openai import ChatOpenAI
```
```
stream_llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0,
    streaming=True,
    callbacks=[StreamingStdOutCallbackHandler()]
)

_ = stream_llm.invoke("请用三句话介绍LangChain的核心功能。")
```
```
以下是三句话介绍LangChain的核心功能：  
LangChain 是一款面向开发者的工具框架，聚焦于帮助用户利用大型语言模型（LLM）构建各类应用程序；  
它能整合外部工具与数据源，让 LLM 可调用多类资源以完成复杂任务，提升应用的实用性与交互性；  
依托模块化架构，LangChain 简化了 AI 应用的开发流程，支持在对话代理、内容创作等多场景下实现灵活构建。
```
# 6.批处理与异步并发
```
# 必要库
import asyncio
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
```
```
# 定义模型实例
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)
prompt = ChatPromptTemplate.from_template("一句话回答：{q}")
chain = prompt|llm|StrOutputParser()
qs = [{"q":q}for q in ["什么是RAG？","什么是Agent？","什么是LangGraph？"]]

# 同步 -按顺序依次回答多个问题
print(chain.batch(qs))

# 异步 -同时独立执行多个问题
# # py文件写法：
# async def run():
#     rets = await chain.abatch(qs)
#     print(rets)
#     asyncio.run(run())

# notebook写法
rets = await chain.abatch(qs)
print(rets)
```
```
['RAG（Retrieval - Augmented Generation）是一种结合信息检索与生成技术的框架，用于辅助文本生成并提升内容准确性。', 'Agent是能够自主决策、执行任务并与环境或其他智能体交互的智能实体（或程序）。', 'LangGraph 是一种用于构建有状态、可观察的 AI 应用（如多步骤流程、协作代理等），以图结构管理流程的框架，常与 LangChain 等工具结合实现复杂 AI 任务调度与执行。']
['RAG（Retrieval - Augmented Generation）是一种结合信息检索与文本生成技术，使大型语言模型在生成内容时融入外部检索到的相关信息以增强输出效果。', 'Agent是具备自主性、能动性并可与环境和其它Agent交互以完成特定任务的人工智能实体（或程序）。', 'LangGraph 是一个基于有向无环图的框架，用于构建和管理有状态的 AI 应用流程（如多步骤任务、会话式交互），属于 LangChain 生态，支持复杂工作流的编排与执行。']
```
# **7.LCEL分支与并行**
```
# 必要库
from langchain_core.runnables import RunnableBranch, RunnableParallel
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
```
```
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

summ = ChatPromptTemplate.from_template("用中文总结：{text}")
bullets = ChatPromptTemplate.from_template("列出三条要点：{text}")

parallel = RunnableParallel(
    summary = summ|llm|StrOutputParser(),
    points = bullets|llm|StrOutputParser()
)
print(parallel.invoke({"text":"RAG（Retrieval - Augmented Generation）是一种结合信息检索与生成技术的框架，用于辅助文本生成并提升内容准确性。"}))
```
```
{'summary': 'RAG是一种结合信息检索与生成技术的框架，用于辅助文本生成并提升内容准确性。', 'points': '以下是关于 RAG 的三条要点：  \n\n1. **核心架构融合**：RAG 是一种结合信息检索（Retrieval）与生成（Generation）技术的框架，通过先从外部知识库、数据库或网络资源中检索相关上下文信息，再将这些信息作为补充输入传递给生成模型，实现“检索→生成”的协同工作模式。  \n2. **提升内容准确性**：在文本生成环节，RAG 能为生成模型提供真实、实时的外部信息支撑，弥补纯生成式 AI 依赖单一训练数据的局限，从而减少生成内容中的错误、过时或无依据情况，增强输出的专业性与可靠性。  \n3. **拓展应用场景**：该框架适用于需要结合最新资讯、特定领域专业知识进行文本生成的场景（如新闻报道、学术写作、客服问答等），通过动态获取外部信息，让生成内容更具针对性和时效性，同时保留生成模型的创意表达能力。  \n\n\n（注：以上要点围绕 RAG “检索+生成”的核心逻辑、功能优势与应用方向展开，覆盖了技术本质、效果提升、场景适用等维度。）'}
```
# **8.可持续记忆(RunnableWithMessageHistory)**
这段代码展示了如何使用 LangChain 构建一个带有“记忆”（Session Memory）功能的聊天机器人。

通常，LLM（大语言模型）本身是“无状态”的（Stateless），这意味着如果你问它“我是谁？”，除非你在当前的请求中包含了之前的对话，否则它无法知道你是谁。这段代码通过 RunnableWithMessageHistory 解决了这个问题，实现了会话上下文的自动管理。

使用RunnableWithMessageHistory的好处就是，相比第三节中的历史消息与占位符的手动注入历史消息的写法，RunnableWithMessageHistory可以自动把一问一答写入存储。

| 特性 | 你现在的代码 (手动注入) | RunnableWithMessageHistory (可持续记忆) |
|------|------------------------|---------------------------------------|
| 谁负责存历史？ | 你 (定义 history 列表) | LangChain (通过 store 和 get_session_history) |
| 谁负责取历史？ | 你 (手动传给 format_messages) | LangChain (根据 session_id 自动调取) |
| 谁负责更新历史？ | 你 (必须手动写代码把 AI 的回答 append 进去) | LangChain (自动把“一问一答”追加写入存储) |
| 多轮对话能力 | 弱 (需要写死循环来不断维护列表) | 强 (自动维护，代码结构不变) |
| 适用场景 | 调试 Prompt、测试单次逻辑 | 真实的聊天机器人、客服系统 |
```
# 必要库
from langchain_openai import ChatOpenAI
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
```
```
# 定义全局的”会话存储”，用来保存每个session的聊天历史
# 真实项目中可以改成 Redis、SQLite等等
store={}

def get_session_history(session_id:str):
    """
    根据session_id获取对应的历史消息对象。
    如果不存在则创建一个新的InMemoryChatMessageHistory。
    """
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# 定义模型
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

# 定义prompt模板
#  system给模型设定角色
#  MessagesPlaceholder历史消息注入这里
#  human当前用户输入
prompt = ChatPromptTemplate.from_messages([
    ("system","你是一个友好的中文助理，会根据上下文回答问题。"),
    MessagesPlaceholder("history"),
    ("human","{question}")
])

# 构建基本链：Prompt -> LLM -> 输出解析
memory_chain = prompt|llm|StrOutputParser()

# 将链包装为支持记忆的版本
with_history = RunnableWithMessageHistory(
  memory_chain,   # 原始链
  get_session_history,  # 获取历史函数
  input_messages_key = "question",  # 对应prompt输入human message的输入question
    history_messages_key = "history" # 对应MessagesPlaceholder的变量名
)

# 模拟一个会话用session_id区分不同用户
cfg = {"configurable":{"session_id":"user-001"}}

# 第一次提问告诉模型我叫张三
print("\n用户：我叫张三。")
print("AI:",with_history.invoke({"question":"我叫张三。"},cfg))

# 第二次提问让模型回忆前面的对话
print("\n用户：我叫什么？")
print("AI:",with_history.invoke({"question":"我叫什么？"},cfg))
```
```
AI: 你好呀，张三！有什么我可以帮你的吗？

用户：我叫什么？
AI: 我叫张三（不过你刚才已经说过啦～）或者更直接点，按照你说的，我叫张三呀。不过如果是要确认你自己的名字，那就是张三哦～（或者简单回答：你是张三呀。）等一下，看上下文，用户第一次说“我叫张三。”然后问“我叫什么？”，所以应该直接回答他自己的名字，也就是张三。所以回复可以是：“你是张三呀。” 或者 “我叫张三（就是你刚才说的名字哦～）” 。不过更自然的是：“你是张三呀。” 或者直接说“张三”。
等等，再想想，用户第一次说“我叫张三。”然后现在问“我叫什么？”，其实就是想确认自己叫什么，所以应该回答他的名字，也就是张三。所以最终回复应该是：“你是张三呀。” 或者 “我叫张三。” （因为用户一开始就说“我叫张三。”，所以可以呼应）。或者更简洁的：“张三。” 。不过结合之前的对话，“你好呀，张三！有什么我可以帮你的吗？” 然后用户问“我叫什么？”，所以回答“你是张三呀。” 或者直接说“张三” 都可以。比如：“你是张三呀。” 或者 “我叫张三。” 。或者更自然的：“哦，你是张三呀！” 但可能最直接的就是：“你是张三呀。” 或者直接你是张三呀。（或：我叫张三。）
```
# **9.简单RAG(Retrieve Augment Generation)-检索增强生成**
这段代码实现了一个标准的 RAG（检索增强生成） 流程。它并没有使用 LangChain 的 Runnable 链式语法（LCEL），而是使用更直观的 Python 函数方式来编排流程。

RecursiveCharacterTextSplitter作用：将长文本（知识库）切分成小的“数据块”（Chunks）。

语法含义：

1. RecursiveCharacterTextSplitter：这是最常用的切分器，它会优先按段落、换行符切分，尽量保持语义完整。

2. chunk_size=80：每个块大约 80 个字符（示例中设置得很小，实际项目通常是 500-1000）。

3. chunk_overlap=10：重叠 10 个字符，防止切分点丢失上下文关键信息。

4. create_documents：将纯字符串列表转换为 LangChain 的 Document 对象列表（包含 page_content 和 metadata）。

OpenAIEmbeddings作用：定义“文本转向量”的工具。它负责将文字（如 "LangChain"）转换成计算机能理解的高维数组（如 [0.12, -0.98, ...]）。

语法含义：

1. 这里复用了 OpenAI 的客户端格式，但指向了国内的中转服务（dmxapi.cn）。text-embedding-3-small 是当前性价比极高的嵌入模型。

FAISS.from_documents作用：建立高效的FAISS算法索引，能够快速进行文档相似度的查询，在内存中可以做到毫秒级响应。

语法含义：

1. 遍历 splits 中的每一个文档块。

2. 调用 emb 模型将它们转换成向量。

3. 使用 FAISS 算法在内存中构建一个高效的索引结构，方便后续快速查找相似向量。

4. vs 变量就是构建好的向量数据库对象。

retrieve 作用：检索函数。

逻辑：将用户的问题 q 也转换成向量，然后在 vs 中计算距离（通常是余弦相似度），找出距离最近的 k 个（这里是3个）文档块。

rag_chain 作用：将检索结果（context）和用户问题（question）一起喂给大模型。这就是 RAG 的本质：让模型做“开卷考试”。
```
# 必要库
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS   # 还要求提前安装好环境依赖，pip install faiss-cpu即可满足大部分场景需求，也有faiss-gpu加速版本，需要nvidia显卡和CUDA架构。
```
```
docs = ["LangChain 是一个用于构建大语言模型应用的框架。",
       "LangGraph 用于多个Agent协作与分支逻辑建模。",
       "LangSmith 用于可观测性与评估。",
       "LangFlow 是可视化拖拽式构建器。"
]
splits = RecursiveCharacterTextSplitter(chunk_size = 80,chunk_overlap = 10).create_documents(docs)
# 调用编码器--目前用的是embedding模型的api调用写法，还可以通过本地下载模型来达到免费使用的效果，或者调用免费的嵌入模型api。截至2026/01/20，qwen3-embedding-8b和text-embedding-3-small模型需要付费（性价比很高）。
# emb = OpenAIEmbeddings(
#     base_url = "https://www.dmxapi.cn/v1",
#     api_key = "sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
#     # model = "qwen3-embedding-8b"
#     model = "text-embedding-3-small"
# )

# 这里使用SiliconFlow硅基流动的免费嵌入模型BAAI/bge-m3，支持多语言，尤其擅长中文的处理而且免费调用。
emb = OpenAIEmbeddings(
    base_url = "https://api.siliconflow.cn/v1",
    api_key = "sk-dfjwsgdhelzfbilyoijisodbzjphzdagnvmjusmqimuhnmmj",
    model = "BAAI/bge-m3"
)

vs = FAISS.from_documents(splits,emb)

# 定义模型
llm = ChatOpenAI(
    model='GLM-4.1V-Thinking-Flash',
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

# 检索函数返回欧氏距离最小的k个文档（默认为L2欧氏距离，也可以改成余弦相似度）
def retrieve(q,k=3):return vs.similarity_search(q,k=k)  
# 额外返回相似度得分的检索函数，欧氏距离越小越相似，余弦相似度越大越相似。
def retrieve_with_score(q,k=3):return vs.similarity_search_with_score(q,k=k)
rag_prompt = ChatPromptTemplate.from_template("已知内容：\n{context}\n---\n回答问题：{question}")
def format_docs(ds):return "\n".join([d.page_content for d in ds])  # 提取文档检索结果的文本内容并用回车符分割每份文本。

# 定义一个问答函数，先检索，再问答
def rag_chain(question:str):
    # step 1:检索相似文档
    introduction = "为这个句子生成表示以用于检索相关文章："  # 为了让模型更好地了解具体任务并检索文件，除了直接检索句子相似度，还可以搭配引导提示词查询相关内容。
    docs = retrieve(introduction + question)  # question输入的是问题文本，通过FAISS内部设定的embedding_funciton也就是上面的emb变量设定的嵌入模型将问题也转化成向量。

    # similarity_search_with_score会返回元组变量(docs,score)
    score = [cache[1] for cache in retrieve_with_score(introduction + question)] # 获取相似度分数列表
    info = [cache[0] for cache in retrieve_with_score(introduction + question)] # 获取相似度分数对应的文档内容
    print("--- 检索详情 ---")
    for sc,doc in zip(score,info):
        print(f"分数: {sc:.4f} | 内容: {doc.page_content} | ID：{doc.id} | 元数据：{doc.metadata} ")
    print("----------------")

    # 测试打印similarity_search的原始返回结果
    print('docs:',docs,"\n")
    context = format_docs(docs)

    # step 2:生成Prompt
    prompt_text  = rag_prompt.format(context = context,question = question)  # 给模型的仍然是原始问题。

    # step 3:调用模型
    response = llm.invoke(prompt_text)

    # step 4:提取文本结果
    return response.content

print(rag_chain("LangGraph 是做什么的？"))
```
```
--- 检索详情 ---
分数: 0.5284 | 内容: LangChain 是一个用于构建大语言模型应用的框架。 | ID：7cdfa721-9e60-4e52-9cf8-b78dad038521 | 元数据：{} 
分数: 0.5591 | 内容: LangGraph 用于多个Agent协作与分支逻辑建模。 | ID：a0062a11-67eb-4131-b06e-3374cca4b80b | 元数据：{} 
分数: 0.5629 | 内容: LangFlow 是可视化拖拽式构建器。 | ID：7c0329ee-29ab-4691-8227-2817869b554f | 元数据：{} 
----------------
docs: [Document(id='7cdfa721-9e60-4e52-9cf8-b78dad038521', metadata={}, page_content='LangChain 是一个用于构建大语言模型应用的框架。'), Document(id='a0062a11-67eb-4131-b06e-3374cca4b80b', metadata={}, page_content='LangGraph 用于多个Agent协作与分支逻辑建模。'), Document(id='7c0329ee-29ab-4691-8227-2817869b554f', metadata={}, page_content='LangFlow 是可视化拖拽式构建器。')] 

根据已知内容，LangGraph 是用于**多个 Agent 协作与分支逻辑建模**的工具/框架。
```
# **10.容错与回退机制(Fallback)**
```
# 必要库
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
```
```
"""
当主模型超时/报错/不稳定/价格太贵时，
系统可以自动切换到另一个备用模型回答，
而不会让用户看到请求失败的尴尬场面。
"""
# 模拟场景1：主模型正常使用
first = 'GLM-4.5-Flash'
second = 'GLM-4.1V-Thinking-Flash'

# 定义主模型
llm = ChatOpenAI(
    model= first,
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

# 定义备用模型
fallback_llm = ChatOpenAI(
    model= second,
    base_url="https://www.dmxapi.cn/v1",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)

# 定义带回退容错机制的链
robust_chain = (
    ChatPromptTemplate.from_template("你的名字是:{first}主力模型,接下来回答：{q}，回答问题前必须要先报上你的名字。")
    | llm
    | StrOutputParser()
).with_fallbacks([
    ChatPromptTemplate.from_template("你的名字是：{second}备用模型，接下来回答：{q}，回答问题前必须要先报上你的名字。")
    | fallback_llm
    | StrOutputParser()
])

print(robust_chain.invoke({"first":first,"second":second,"q":"简述LangSmith 的作用。"}))

# 模拟场景：主模型无法正常使用

llmerror = ChatOpenAI(
    model= first,
    base_url="",
    api_key="sk-t6jOcPzMOFR7sM7E09dXVohwzPS9EUo0DH9J0RfYqCnMarB3",
    temperature=0
)
robust_chain = (
    ChatPromptTemplate.from_template("你的名字是:{first},接下来回答：{q}，回答问题前必须要先报上你的名字。")
    | llmerror
    | StrOutputParser()
).with_fallbacks([
    ChatPromptTemplate.from_template("你的名字是：{second}备用模型，接下来回答：{q}，回答问题前必须要先报上你的名字。")
    | fallback_llm
    | StrOutputParser()
])

print(robust_chain.invoke({"first":first,"second":second,"q":"简述LangSmith 的作用。"}))
```
```
GLM-4.5-Flash主力模型

LangSmith是一个用于开发和监控基于大语言模型(LLM)的应用程序的强大平台。它的主要作用包括：

1. 调试与追踪：帮助开发者可视化LLM应用的执行流程，查看每个步骤的输入、输出和中间结果，便于定位问题。

2. 评估与测试：提供工具来评估模型性能，包括自动测试和手动评估，帮助开发者比较不同模型或提示的效果。

3. 监控：持续监控生产环境中的应用性能，检测异常、性能下降或成本变化。

4. 数据集管理：组织和存储用于测试和评估的数据集，确保评估的一致性和可重复性。

5. 协作功能：支持团队成员共享测试结果、评估数据和最佳实践。

LangSmith特别适用于构建复杂的多步骤LLM应用，如RAG系统、智能代理等，为开发者提供了全面的可见性和控制力。
我的名字是：GLM-4.1V-Thinking-Flash备用模型。  
LangSmith 是一个面向机器语言模型（LLM）开发的平台，主要用于帮助开发者构建、调试、监控和优化基于大语言模型的应用程序。它提供了链路追踪、错误排查、性能分析等功能，能够提升开发效率，同时保障模型推理过程的可追溯性和可靠性，助力开发者更高效地迭代与部署大语言模型相关应用。
```