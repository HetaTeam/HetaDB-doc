Welcome to HetaDB's documentation!
===================================

**帮助用户轻松构建知识库和从知识库中获得可信准确回答！** (`前往Github <https://github.com/HetaTeam/HetaDB/>`_)

用户可以通过配置必要的数据地址信息，HetaDB就可以自动化地解析数据，并根据数据不同的存取方式使用合适的数据库，然后根据用户的查询请求来便捷定位知识，并整合生成回答

1. HetaDB的核心功能
-------------------------

* 多种主流文档的解析器
* 多类型的数据库的部署和使用
* 多种信息检索方式的集成调用
* 知识库文档名目管理

2. 大语言模型的API支持
----------------------

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
   :maxdepth: 3
   :caption: 快速上手
   :name: quick_start

   quick_start/quick_start.rst

   quick_start/config.rst

.. toctree::
   :maxdepth: 2
   :caption: 文件解析
   :name: file_parsing

   file_parsing/_parser_assignment_module
   file_parsing/_unified_output_module
   file_parsing/_text_parser_module
   file_parsing/_doc_parser_module
   file_parsing/_html_parser_module
   file_parsing/_image_parser_module
   file_parsing/_sheet_parser_module


.. toctree::
   :maxdepth: 2
   :caption: 数据库集成
   :name: database_integration

   db_build/db_build.rst
   db_build/_graph_db_module.rst
   db_build/_vector_db_module.rst
   db_build/_auto_schema_csv_ingestor_module.rst


.. toctree::
   :maxdepth: 2
   :caption: 数据库集成
   :name: database_integration

   db_build/db_build.rst
   db_build/_graph_db_module.rst
   db_build/_vector_db_module.rst
   db_build/_auto_schema_csv_ingestor_module.rst


.. toctree::
   :maxdepth: 2
   :caption: 检索与生成
   :name: retrival_generation

   query/_optimized_kb_query_module.rst
   query/_descriptive_t2s_engine_module.rst
   query/_knowledge_api_module.rst



.. toctree::
   :maxdepth: 2
   :caption: 优化生成质量  
   :name: organization

   organize/_query_rewriter_module.rst
   organize/_multi_hop_qa_module:.rst


.. toctree::
   :maxdepth: 2
   :caption: 额外工具  
   :name: organization

   ex_tools/_bookmarks_archiver_module.rst
   ex_tools/_email_archiver_module.rst

