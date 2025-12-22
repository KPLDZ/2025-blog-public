# BiliBiliLoader的局限
在上一篇文章中，我们通过BiliBiliLoader创建了一个纯文本处理的大语言模型文本总结工作流，本质上我们通过该工具获取的是视频的字幕信息和元数据（标题、简介等），而非视频、音频，所以对一些自带字幕，或者作者已经上传过字母文档`transcript`的视频,可以非常精准地总结视频内容；如果视频本身不带字幕信息，BiliBiliLoader返回的`transcript`就会为空，返回的总结内容就会驴头不对马嘴。因此，我修改了上一版代码，优化了给模型的上文信息，根据是否成功获取字幕信息而采取两种总结策略：

1. 如果获取到了字幕信息，按照原有逻辑正常进行深度总结。
2. 如果未获取到字幕信息，则根据视频元数据信息（标题/简介等）进行粗略的内容推测和总结。

# 库导入与初始化配置
第二版代码的环境与第一版完全一致，链接直达：`https://2025-blog-public-taupe-ten.vercel.app/blog/LLM-Bilibili-vedio`

第二版代码在该部分主要修改了`default_prompt`，也就是提示词，以适配两种策略。

```
# 导入所需的、兼容 LangChain v1.x 的库
from langchain_community.document_loaders.bilibili import BiliBiliLoader
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from typing import List
from langchain_core.documents import Document

# ======================================================================
# ❗ Bilibili Cookie 配置 ❗
# ======================================================================
SESSDATA ="1cd1be3e%2C"
BUVID3 = "74841"
BILI_JCT = "8398e"


# 1. 配置 LLM (适配 DeepSeek API)
# DeepSeek API 配置
api_base = "https://api.deepseek.com" 
api_key = "sk-c2" # ⚠️ 请替换为您的 DeepSeek API Key
model_name = "deepseek-chat" # DeepSeek 官方推荐的模型名称

llm = ChatOpenAI(
    openai_api_key=api_key, 
    openai_api_base=api_base, 
    model_name=model_name
)

# 2. 定义总结提示词模板
default_prompt = """你是一个专业的视频内容分析师。
你的任务是根据提供的视频信息（可能包括标题、简介和字幕文稿）进行深度总结。

【重要规则】：
- 如果提供的信息中包含详细的“视频字幕文稿”，请以文稿为准进行深度分析。
- 如果“视频字幕文稿”处标注为[未获取到有效字幕]，说明该视频可能没有语音或抓取失败。此时请务必利用“视频标题”和“视频简介”进行推测性总结。
- 在这种情况下，请在总结开头提示用户：“由于未获取到字幕，以下总结基于视频元数据生成。”
"""

summary_prompt = ChatPromptTemplate.from_messages([
    ("system", default_prompt),
    ("human", "视频基本信息与文本内容:\n\n{context}\n\n问题/指令: {question}")
])
```

# Chain 构建与核心函数定义¶

该部分修改了`format_all_docs`和`summarize_video`函数，此次改动完全不影响原来的单个视频总结的函数调用方式；`summarize_multiple_videos`函数完全未修改，同样支持原来的多视频总结的调用方式，

```
def format_all_docs(docs: List[Document]) -> dict:
    """
    解析 Document 列表。
    返回一个字典，包含：
    - has_content: 布尔值，表示是否有有效字幕
    - full_context: 组合后的上下文（标题+简介+字幕）
    """
    if not docs:
        return {"has_content": False, "full_context": ""}

    # 提取元数据
    metadata = docs[0].metadata
    title = metadata.get('title', '未知标题')
    description = metadata.get('description', '无简介')
    
    # 提取并合并所有字幕内容
    content_parts = [doc.page_content for doc in docs if doc.page_content.strip()]
    full_content = "\n".join(content_parts)
    
    # 判断是否真的有字幕
    has_content = len(full_content.strip()) > 0
    
    # 构造喂给 AI 的总上下文
    context_str = f"【视频标题】：{title}\n"
    context_str += f"【视频简介】：{description}\n"
    context_str += f"【视频字幕文稿】：\n{full_content if has_content else '[未获取到有效字幕]'}"
    
    return {
        "has_content": has_content,
        "full_context": context_str
    }


# 核心总结函数
def summarize_video(bvid: str, question: str):
    """
    增强版总结函数：具备字幕检测与分级总结功能。
    """
    print(f"--- 正在处理视频: {bvid} ---")
    
    try:
        # 1. 加载文档
        video_urls = [f"https://www.bilibili.com/video/{bvid}"]
        loader = BiliBiliLoader(
            video_urls,
            sessdata=SESSDATA,
            bili_jct=BILI_JCT,
            buvid3=BUVID3,
        )
        documents = loader.load()
        
        # 2. 解析内容与元数据
        data_package = format_all_docs(documents)
        full_context = data_package["full_context"]
        
        # 3. 状态报告
        if data_package["has_content"]:
            print(f"✅ [报告] 视频 {bvid}：已成功获取字幕信息，进行深度总结。")
        else:
            print(f"⚠️ [报告] 视频 {bvid}：字幕信息获取失败，将根据视频元数据（标题/简介）进行粗略总结。")

        # 4. 调用 LLM
        prompt_value = summary_prompt.invoke({
            "context": full_context, 
            "question": question
        })
        
        ai_message = llm.invoke(prompt_value.to_messages())
        ans = ai_message.content 
        
        print(f"--- 视频 {bvid} 处理完成 ---")
        return bvid, ans
        
    except Exception as e:
        error_msg = f"总结失败，发生技术错误: {str(e)}"
        print(f"❌ 处理视频 {bvid} 时出错: {e}")
        return bvid, error_msg

def summarize_multiple_videos(bvid_list: List[str], question: str) -> dict:
    """
    同时总结多个视频。
    """
    results = {}
    print(f"--- 正在处理 {len(bvid_list)} 个视频的总结 ---")
    for bvid in bvid_list:
        bvid_result, summary = summarize_video(bvid, question)
        results[bvid_result] = summary
    return results
```

# 单个视频总结示例

```
# 单个视频 BVID
single_bvid = "BV1ynm3BsE9Q" # 使用的 BV ID
single_question = "请用500字总结这个视频的主要内容、核心观点和关键结论。"

# 执行单视频总结
single_video_summary = summarize_video(single_bvid, single_question)

print("\n--- 单视频总结结果 ---")
print(f"视频 ID: {single_video_summary[0]}")
print("总结内容:")
print(single_video_summary[1])
```

```
--- 正在处理视频: BV1ynm3BsE9Q ---
C:\Users\asus\AppData\Roaming\Python\Python312\site-packages\langchain_community\document_loaders\bilibili.py:131: UserWarning: No subtitles found for video: https://www.bilibili.com/video/BV1ynm3BsE9Q. Returning empty transcript.
  warnings.warn(
⚠️ [报告] 视频 BV1ynm3BsE9Q：字幕信息获取失败，将根据视频元数据（标题/简介）进行粗略总结。
--- 视频 BV1ynm3BsE9Q 处理完成 ---

--- 单视频总结结果 ---
视频 ID: BV1ynm3BsE9Q
总结内容:
由于未获取到字幕，以下总结基于视频元数据生成。

根据视频标题《这一年，我们在B站看了些啥？》，可以推测这是一部年度回顾或盘点类视频。其核心内容极有可能是对过去一年（具体年份需结合视频发布时间判断）在Bilibili（B站）平台上，广大用户观看内容的系统性梳理、总结与呈现。

视频的主要内容和核心观点可能围绕以下几个方面展开：
1.  **年度热门内容盘点**：视频很可能通过数据可视化（如图表、动画）等形式，展示该年度B站最受关注的热门视频类别、播放量最高的UP主、现象级的“爆款”视频或系列。这可能涵盖动画、游戏、知识、生活、音乐、鬼畜等B站特色分区。
2.  **社区文化与趋势洞察**：分析该年度在B站兴起的流行文化趋势、网络热梗、用户共创内容（如二创、弹幕文化）以及社区讨论热点。核心观点可能在于展现B站作为年轻文化社区的活力、创造力和独特性。
3.  **用户行为与情感共鸣**：通过展示代表性的弹幕、评论或用户故事，总结这一年B站用户的集体记忆、情感共鸣点（如对特定作品的热爱、对社会事件的关注、共同的青春回忆等）。关键结论可能指向平台与用户之间深厚的情感连接。
4.  **平台发展与展望**：视频或许会简要提及B站该年度在内容生态、技术功能或社区建设方面的重大进展，并以此为基础，对未来的内容趋势或社区发展方向进行展望。

**关键结论**（推测性）：该视频旨在通过年度盘点，不仅呈现数据层面的观看“是什么”，更试图解读背后“为什么”——即揭示驱动这些观看行为的社会文化心理、社区归属感以及时代情绪。它最终塑造的是一种“我们共同经历”的集体叙事，强化B站用户对平台的认同感和作为社区一员的参与感。
```

# 多视频串行排队总结

```
# 1. 准备要总结的视频列表 
bvid_list_to_process = [
    "BV15pqUBpELS",  # 视频 A
    "BV1xEm1B9E56",  # 视频 B 
    "BV1BnqrB6ED3"   # 视频 C 
]

# 2. 定义统一的总结指令/问题
# 这个指令会被传递给每一个视频，要求 LLM 执行相同的总结任务
common_question = "请用三个核心要点概括本视频的内容，每个要点不超过30字，并指出视频的最终结论。"
# common_question = "请用500字总结这个视频的主要内容、核心观点和关键结论。"  # 或者用刚才一样的输入。

# 3. 调用多视频总结函数
# results 是一个字典，键是 BV 号，值是 LLM 返回的总结文本
multiple_video_results = summarize_multiple_videos(
    bvid_list=bvid_list_to_process,
    question=common_question
)

# 4. 打印结果
print("\n--- 多视频总结报告 ---")
for bvid, summary in multiple_video_results.items():
    print(f"\n>>>> 视频 ID: {bvid} <<<<")
    print("---------------------------------")
    print(summary)
    print("---------------------------------")
```

```
--- 正在处理 3 个视频的总结 ---
--- 正在处理视频: BV15pqUBpELS ---
⚠️ [报告] 视频 BV15pqUBpELS：字幕信息获取失败，将根据视频元数据（标题/简介）进行粗略总结。
--- 视频 BV15pqUBpELS 处理完成 ---
--- 正在处理视频: BV1xEm1B9E56 ---
✅ [报告] 视频 BV1xEm1B9E56：已成功获取字幕信息，进行深度总结。
--- 视频 BV1xEm1B9E56 处理完成 ---
--- 正在处理视频: BV1BnqrB6ED3 ---
✅ [报告] 视频 BV1BnqrB6ED3：已成功获取字幕信息，进行深度总结。
--- 视频 BV1BnqrB6ED3 处理完成 ---

--- 多视频总结报告 ---

>>>> 视频 ID: BV15pqUBpELS <<<<
---------------------------------
由于未获取到字幕，以下总结基于视频元数据生成。

**核心要点：**
1.  视频探讨日本京都大学因一次系统更新导致的数据丢失事件。
2.  事件过程可能涉及更新操作失误、备份失效或系统兼容性问题。
3.  分析此类事故对科研机构数据安全的警示与教训。

**最终结论：**
基于标题推断，视频结论很可能强调系统更新与数据备份的重要性，警示机构需完善运维流程以防止灾难性数据损失。
---------------------------------

>>>> 视频 ID: BV1xEm1B9E56 <<<<
---------------------------------
【核心要点概括】  
1. **双星系统加速死亡**：距离地球7800光年的双星（白矮星与伴星）正疯狂互相吞噬，轨道周期持续缩短。  
2. **将引发超新星爆发**：预计几十年至本世纪末发生新星爆发，最终合并产生亮度超太阳50亿倍的超新星。  
3. **对地球无威胁**：7800光年距离足够安全，爆发时白天可见亮星，但不会影响地球生态。  

【最终结论】  
人类可能在几十年内亲眼见证一次史无前例的超新星爆发（白天可见），这既是天文学研究的重要机遇，也是一场安全的宇宙奇观，值得所有人期待。
---------------------------------

>>>> 视频 ID: BV1BnqrB6ED3 <<<<
---------------------------------
1. **游戏设计争议**：SUPERHOT VR要求玩家模拟自杀进入游戏，引发伦理争议与设计师的自我批判。  
2. **剧情深度解析**：游戏通过隐藏彩蛋和叙事，暗示其可能是利用人脑的杀人程序或对抗AI的工具。  
3. **自毁主题贯穿**：系列核心是主角不可抑制的自毁倾向，从沉迷到自我毁灭的循环。  

**最终结论**：视频认为SUPERHOT系列的核心并非逻辑完美的故事，而是通过“自毁”主题连接玩法与叙事，并反思游戏设计在真实与虚拟边界引发的伦理问题。设计师删除争议内容的行为本身，也成为了这种“自毁”表达的延伸。
---------------------------------
```
