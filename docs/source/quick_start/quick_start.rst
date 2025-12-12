.. _quick_start_guide:

快速入门指南
============

本指南将帮助您在 **5 分钟内** 完成 HetaDB 系统的环境搭建、依赖安装与首个示例运行

.. raw:: html

    <div class="quick-start">
        <h3>⚡ 5 分钟快速体验</h3>
        <p>无需复杂配置，三步即可运行完整流程：文件解析 → 数据入库 → 智能问答。</p>
    </div>

环境准备
--------

系统要求
^^^^^^^^

- **操作系统**：Linux / macOS / Windows（推荐 Linux）
- **Python 版本**：3.10 或更高（不支持 3.9 及以下）
- **内存**：≥ 8 GB RAM（处理大文档时建议 ≥ 16 GB）
- **存储**：≥ 10 GB 可用空间（用于模型缓存、中间文件与数据库）
- **GPU（推荐）**：NVIDIA GPU（CUDA 支持），用于加速嵌入计算与 LLM 推理  
  > **无 GPU 也可运行**：系统会自动回退到 CPU 模式（速度较慢）

安装步骤
^^^^^^^^

1. **创建虚拟环境**（推荐使用 ``uv``，速度更快）

   .. code-block:: bash

      # 安装 uv（现代 Python 包管理器）
      pip install --upgrade pip
      pip install uv

      # 创建并激活虚拟环境
      uv venv h-rag --python=3.10
      source h-rag/bin/activate      # Linux / macOS
      # 或
      h-rag\Scripts\activate         # Windows

   > **备选方案**：使用 conda
   >
   > .. code-block:: bash
   >
   >    conda create -n h-rag python=3.10 -y
   >    conda activate h-rag

2. **安装 HRAG 及其依赖**

   .. code-block:: bash

      # 以可编辑模式安装项目（开发模式）
      uv pip install -e .

   > 此命令将自动安装：
   > - 核心依赖（PyTorch、Transformers、LangChain）
   > - 向量数据库（Milvus / pymilvus）
   > - 数据库驱动（psycopg2、asyncpg）
   > - 解析工具（pdfminer、beautifulsoup4、pandas 等）

3. **验证安装**

   .. code-block:: bash

      # 验证 PyTorch
      python -c "import torch; print('✅ PyTorch', torch.__version__, '| CUDA available:', torch.cuda.is_available())"

      # 验证 Transformers
      python -c "import transformers; print('✅ Transformers', transformers.__version__)"

      # 验证 HRAG 模块
      python -c "from src.file_parsing import ParserAssignment; print('✅ HRAG core imported successfully')"

   > 若无报错且输出版本信息，则安装成功。

运行示例
--------

HRAG 提供完整的端到端示例，涵盖 **文件解析 → 数据入库 → 问答服务** 三大环节。

1. **准备与解析原始文件**

   .. code-block:: bash

      # 执行文件解析流水线（PDF/DOC/HTML/表格等）
      python examples/ex_datafactory.py

   > 输出将生成：
   > - 哈希重命名的原始文件
   > - 结构化 JSONL（文本、图像、表格）
   > - 中间目录：``data/parsed_file/``

2. **向量与结构化数据入库**

   .. code-block:: bash

      # 将解析结果导入 PostgreSQL + Milvus
      python examples/ex_database_ingestion.py

   > 此步骤将：
   > - 在 PostgreSQL 中创建图谱表（节点/关系）
   > - 在 Milvus 中插入文本块与图谱向量
   > - 生成表结构描述（用于 Text2SQL）

3. **启动问答服务并查询**

   .. code-block:: bash

      # 启动 FastAPI 服务（默认端口 8000）
      python examples/ex_query.py

      # 然后在另一个终端发送请求
      curl -X POST http://localhost:8000/api/knowledge/query \
           -H "Content-Type: application/json" \
           -d '{
                 "query": "项目报告中的关键技术有哪些？",
                 "kb_id": 1,
                 "user_id": "demo",
                 "mode": 0
               }'

   > 支持三种检索模式：
   > - ``mode=0``：基础检索（默认）
   > - ``mode=1``：查询改写
   > - ``mode=2``：多跳推理

4. **探索更多示例**

   - ``examples/ex_image_caption.py``：图像描述生成
   - ``examples/ex_sheet_analysis.py``：表格语义分析
   - ``examples/ex_multi_hop_qa.py``：复杂多跳问答

