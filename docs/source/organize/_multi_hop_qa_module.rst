.. _multi_hop_qa_module:

多跳问答代理
=============

本模块实现基于 **ReAct（Reason + Act）** 框架的多跳问答能力，通过大语言模型（LLM）**主动调用知识库检索工具**，分步收集信息、验证充分性，并最终生成答案。

适用于复杂、需多步推理的问题（如“华为在5G领域的主要竞争对手有哪些？”），显著优于单次检索+生成的基线方法。

核心机制
--------

- **ReAct 循环**：LLM 在每轮生成“思考 → 动作（调用工具） → 观察”；
- **工具调用**：集成 ``KnowledgeQueryTool``，可多次查询知识库；
- **信息提炼**：
  - **Stage 1**：从检索结果中提取与问题相关的有用信息；
  - **Stage 2**：判断当前累积信息是否足以回答问题；
- **早停机制**：一旦信息充分，立即生成最终答案，避免冗余查询；
- **记忆跟踪**：将提取的有用信息存入 ``memory``，供后续推理使用。

架构图
------

::

   用户问题
      │
      ▼
   LLM（HAgent）
      │
      ├── 思考：是否需要检索？ → 是
      │        │
      │        ▼
      │    调用 KnowledgeQueryTool
      │        │
      │        ▼
      │    执行 perform_knowledge_query
      │        │
      │        ▼
      │    返回检索结果（文本+生成草稿）
      │        │
      │        ▼
      ├── Stage 1：提取有用信息 → 加入 memory
      │        │
      │        ▼
      └── Stage 2：判断信息是否充分 → 是 → 生成 Final Answer

工具：KnowledgeQueryTool
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. autoclass:: src.organize.multi_hop_qa.KnowledgeQueryTool
   :members:
   :undoc-members:

**功能** ：将知识库查询封装为 LLM 可调用的工具。

**输入参数** （JSON 或字符串）：

- ``query`` （必填）：自然语言问题；
- ``top_k``、``kb_id``、``user_id`` 等（可选）：透传给 ``perform_knowledge_query``。

**输出格式**：

.. code-block:: text

   Knowledge query succeeded.
   Answer Draft: <LLM生成的草稿>
   KB Info: {查询元数据}
   Top Hits:
   1. score=0.95 | content=...
   2. score=0.88 | content=...

**设计亮点**：自动处理异步调用（通过线程 + asyncio），兼容同步 LLM 框架（Qwen-Agent）。

代理：HAgent
------------

.. autoclass:: src.organize.multi_hop_qa.HAgent
   :members:
   :private-members:

**继承自**：``qwen_agent.agents.fncall_agent.FnCallAgent``

**关键增强**：
- 自定义 ReAct 提示词（``SYSTEM_EXPLORER``）；
- 禁止模型拒绝回答（强制探索）；
- 集成信息提炼与答案判断子模块（``observation_information_extraction``, ``critic_information``）；
- 支持最大动作次数限制（``action_count``）。

提示词设计
----------

1. **主代理提示（SYSTEM_EXPLORER）**

   强制使用 ReAct 格式，要求每步必须执行动作，禁止拒绝。

2. **信息提炼提示（STSTEM_CRITIIC_INFORMATION）**

   要求模型判断观察结果是否对问题有用，并提取相关信息（JSON 格式）。

3. **答案判断提示（STSTEM_CRITIIC_ANSWER）**

   要求模型判断累积信息是否足以回答问题，若是则生成最终答案（JSON 格式）。

使用示例
--------

API 调用
~~~~~~~~

.. code-block:: python

   from src.organize.multi_hop_qa import MultiHopAgent

   agent = MultiHopAgent()
   trace = agent.answer(
       query="华为在5G领域的竞争对手有哪些？",
       top_n=10,
       kb_id=1,
       max_rounds=3  # 最多检索3次
   )
   final_answer = trace[-1].get("answer", "无法回答")

在 FastAPI 中集成
~~~~~~~~~~~~~~~~~~

已在 ``/api/knowledge/query?mode=2`` 中集成，前端只需指定 ``mode=2``。

配置依赖
--------

从 ``config/api_config.yaml`` 加载多跳 LLM 配置：

.. code-block:: yaml

   llm:
     multi_hop:
       model: "qwen3_32b"
       api_key: "EMPTY"
       base_url: "http://10.140.37.71:8001/v1"
       timeout: 120  # 秒

**注意**：超时值设为 ``null`` 表示无限制（YAML 中写为 ``timeout:``）。

推理流程详解
------------

1. **初始化**：构造 HAgent，注入查询与最大轮数；
2. **第一轮**：
   - LLM 生成思考：“需要查询华为的5G专利”；
   - 调用 ``knowledge_query`` 工具；
   - 获取结果：“华为5G专利涉及技术A、B...”；
   - 提取有用信息 → ``memory = ["华为5G技术A..."]``；
3. **第二轮**：
   - LLM 基于 ``memory`` 生成新思考：“还需查询其他公司的5G专利”；
   - 再次调用工具，查询“爱立信 5G专利”；
   - 提取信息 → ``memory = ["华为...", "爱立信..."]``；
4. **判断**：若 ``critic_information`` 返回 ``judge=true``，则生成最终答案。

异常处理
--------

- **工具调用失败**：返回错误字符串，LLM 可据此调整策略；
- **JSON 解析失败**：重试 10 次（指数退避）；
- **LLM 无动作**：跳过当前轮次，继续循环；
- **超时**：由 OpenAI 客户端抛出，上层捕获。

输出格式
--------

返回推理轨迹（``list[dict]``），每项包含：

- ``thoughts``：LLM 的思考过程；
- ``memory``：当前累积的有用信息；
- ``answer``：最终答案（仅最后一项）。

示例：

.. code-block:: python

   [
     {"thoughts": "需要查询华为5G专利..."},
     {"memory": "Memory:\n- 华为5G技术涉及大规模MIMO..."},
     {"thoughts": "还需查询竞争对手..."},
     {"memory": "Memory:\n- 华为...\n- 爱立信5G专利..."},
     {"answer": "Final Answer: 华为在5G领域的主要竞争对手包括爱立信、诺基亚..."}
   ]

注意事项
--------

- **性能**：每轮需 2 次 LLM 调用（主代理 + 信息提炼/判断），总耗时 = 轮数 × 3 次调用；
- **Token 消耗**：memory 会随轮次增长，长问题需注意上下文长度；
- **工具依赖**：必须实现 ``perform_knowledge_query`` 并返回结构化结果；
- **非确定性**：LLM 行为受 temperature 影响，生产环境建议设置 seed。

典型应用场景
------------

- **多实体比较**：“A 和 B 在 X 方面有何异同？”
- **因果推理**：“为什么 A 会导致 B？”
- **信息聚合**：“总结近三年关于 X 的研究进展”
- **专利/法律分析**：“某技术是否侵犯了专利 Y？”
