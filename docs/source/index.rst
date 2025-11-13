Welcome to HetaDB's documentation!
===================================

HetaDB是可以用于帮助用户轻松构建知识库和从知识库中获得可信准确回答的Python library

用户可以通过配置必要的数据地址信息，HetaDB就可以自动化地解析数据，并根据数据不同的存取方式使用合适的数据库，然后根据用户的查询请求来便捷定位知识，并整合生成回答

HetaDB的核心功能包括：
* 多种主流文档的解析器
* 多类型的数据库的部署和使用
* 多种信息检索方式的集成调用
* 知识库文档名目管理

要部署HetaDB需要大语言模型（如果有处理图片的需求，则需要视觉语言模型）的支持，所以用户需要提前准备要可以调用的大语言模型的API支持，包括但不限于：
* GPT-4 / GPT-3.5 / GPT-4o (OpenAI)
* Claude 3.5 Sonnet / Claude 3 Opus / Claude 3 Haiku / Claude 3 Sonnet (Anthropic)
* Gemini 1.5 Pro / Gemini 1.5 Flash / Gemini 2.0 (Google)
* Qwen-Max / Qwen-Plus / Qwen-Turbo / Qwen2 / Qwen3 (Alibaba)
* Doubao Pro / Doubao Lite (ByteDance)
* Moonshot-v1 (Moonshot AI)
如果用户需要构建超大规模的数据库，建议使用自己部署开源的大语言模型，包括但不限于：
* Llama 3.2 / Llama 3.1 / Llama 3 (Meta)
* Qwen / Qwen2 / Qwen3 (Alibaba) 
* InternLM2 / InternLM3 / InternS1 (Shanghai AI Laboratory)
* GLM-4 / GLM-Edge / GLM-Lite (Zhipu AI)
* Mistral Large / Mixtral 8x22B / Mixtral 8x7B (Mistral AI)



.. note::

   HetaDB是我们所研发的Heta知识库的重要组成部分，如果感兴趣的用户还可以查看Heta知识库的其他组成部分，包括: `HetaRAG <https://github.com/KnowledgeXLab/HetaRAG/>`_, HetaMeM, 和 HetaGen

Contents
--------

.. toctree::

   usage
   api

.. toctree::
   :maxdepth: 2
   :caption: 用户指南
   :name: userguide

   quickstart
   database_Installation
   installation
   configuration
   basic_usage
   advanced_usage