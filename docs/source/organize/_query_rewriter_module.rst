.. _query_rewriter_module:

查询改写器
=============

本模块利用大语言模型（LLM）对用户原始查询进行**语义扩展与改写**，生成多个等效或互补的查询变体，从而提升后续检索系统的召回率与鲁棒性。

适用于知识库问答、语义搜索等场景，尤其对模糊、简略或表述多样的自然语言查询效果显著。

功能特点
--------

- **语义扩展**：自动生成同义、近义、分解式查询；
- **格式规范**：强制 LLM 输出结构化 JSON，便于解析；
- **双后端支持**：
  - **vLLM**（默认）：通过 OpenAI 兼容 API 调用；
  - **Ollama**（可选）：通过 LangChain 集成；
- **容错处理**：自动移除 LLM 返回的 ``<think>`` 推理标签，防止 JSON 解析失败；
- **批量处理**：支持单条或批量查询改写。

使用示例
--------

### 基础用法（vLLM 后端）

.. code-block:: python

   from src.organize.query_rewriter import QueryRewriter

   rewriter = QueryRewriter(api_provider="vllm")
   variations = rewriter.rewrite(
       query="华为2023年有多少5G专利？",
       max_variations=3
   )
   print(variations)
   # 输出示例：
   # [
   #   "华为在2023年申请了多少项5G相关专利？",
   #   "2023年华为公司5G专利数量是多少？",
   #   "华为2023年公布的5G发明专利总数？"
   # ]

### 批量改写

.. code-block:: python

   queries = ["苹果市值", "特斯拉电池技术"]
   results = rewriter.batch_rewrite(queries)
   # 返回: [[...], [...]]

核心类
------

.. autoclass:: src.organize.query_rewriter.QueryRewriter
   :members:
   :undoc-members:

构造函数参数
~~~~~~~~~~~~

``model`` : ChatOllama | None  
   可选。若提供，则使用 Ollama 后端；否则默认使用 vLLM（通过 ``BaseVllmProcessor``）。

``system_prompt`` : str  
   系统提示词（默认为 ``QUERY_REWRITE_PROMPT``），定义改写规则与输出格式。

``api_provider`` : str  
   API 提供商（当前仅 ``"vllm"`` 有完整实现）。

主方法
~~~~~~

.. automethod:: src.organize.query_rewriter.QueryRewriter.rewrite

.. automethod:: src.organize.query_rewriter.QueryRewriter.batch_rewrite

内部辅助类
----------

.. autoclass:: src.organize.query_rewriter.BaseVllmProcessor
   :members:
   :private-members:

**说明**：  
该类封装了 vLLM 的 OpenAI 兼容 API 调用，支持结构化输出与非结构化输出。在 ``QueryRewriter`` 中默认使用非结构化模式（因提示词要求 JSON 字符串）。

提示词设计（QUERY_REWRITE_PROMPT）
--------------------------------

系统使用以下固定提示词引导 LLM：

.. code-block:: text

   You are a helpful assistant that generates multiple search queries based on a single input query.
   Perform query expansion. If there are multiple common ways of phrasing a user question or common synonyms for key words in the question, make sure to return multiple versions of the query with the different phrasings.
   If there are acronyms or words you are not familiar with, do not try to rephrase them.
   Return 3 different versions of the question.
   Do not include any other text or explanation.

   ---Output Format Requirements---
   Return the queries as a JSON object with a key "queries" and a list of strings as the value.
   The output should strictly follow this format:
   {
     "queries": [
       "Expanded_Query_1",
       "Expanded_Query_2",
       "Expanded_Query_3"
     ]
   }

> **关键约束**：
> - 仅返回 3 个变体；
> - 不重写不熟悉的缩写或专有名词；
> - 输出必须为纯 JSON，无任何额外文本。

容错与解析
----------

- **推理标签清理**：自动移除 ``<think>...</think>`` 或 ``<think>...`` 片段，避免因 LLM 推理过程破坏 JSON 格式；
- **降级解析**：若 JSON 解析失败，尝试按行分割文本作为备选方案；
- **空结果处理**：改写失败时（如 LLM 无响应），可选择返回空列表（默认）或抛出异常。

依赖项
------

- **Python 包**：
  - ``openai``（vLLM 调用）
  - ``langchain_core`` + ``langchain_ollama``（Ollama 支持，可选）
  - ``tiktoken``（Token 计数）
  - ``json`` / ``re``（解析与清理）
- **配置**：
  - ``src.utils.load_config.get_chat_cfg()``：包含 vLLM 的 ``api_key``、``base_url``、``model``、``timeout``

配置示例（config/api_config.yaml）
----------------------------------

.. code-block:: yaml

   chat_config:
     api_key: "EMPTY"
     base_url: "http://10.140.37.71:8001/v1"
     model: "qwen3_32b"
     timeout: 120

注意事项
--------

- **模型兼容性**：当前逻辑假设 vLLM 返回 **字符串形式的 JSON**，而非结构化对象；
- **Token 限制**：长查询可能导致 LLM 截断，建议输入查询长度适中；
- **专有名词保护**：提示词明确要求不重写不熟悉词汇，适合专利、技术术语等场景；
- **性能**：每次改写需一次 LLM 调用，批量改写为串行（非并发），高吞吐场景可自行扩展。

典型应用场景
------------

1. **提升召回率**：将单查询扩展为多查询，并行检索后融合结果；
2. **处理表述歧义**：如 “苹果” → [“苹果公司”, “水果苹果”]（需模型具备此能力）；
3. **问题分解**：将复合问题拆解为原子问题（如示例中的国籍判断）；
4. **多语言适配**：可扩展支持中英混合查询的规范化。
