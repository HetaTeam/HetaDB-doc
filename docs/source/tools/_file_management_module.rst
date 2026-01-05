.. _file_management_module:

文件记录管理工具
================

本模块用于处理和汇总已解析文件的元数据信息，从JSONL格式的文件记录中提取文件信息，并生成CSV格式的文件记录汇总表。

功能特性
--------

* **自动文件发现**：自动扫描 ``data/parsed_file`` 目录下的JSONL文件
* **双源数据整合**：整合文本解析和图像描述的元数据
* **CSV格式输出**：生成标准CSV格式的文件记录表
* **字段完整性检查**：确保所有必需字段都存在
* **逗号冲突处理**：自动处理CSV格式中的逗号冲突

数据流程
--------

1. **扫描数据目录**：查找 ``data/parsed_file/jsonl`` 和 ``data/parsed_file/image_desc`` 目录
2. **解析JSONL文件**：读取每个JSONL文件中的记录
3. **提取元数据**：从每条记录中提取 ``meta`` 字段
4. **数据整合**：将所有文件记录整合到统一列表
5. **CSV输出**：生成包含所有文件信息的CSV文件

输出数据结构
------------

生成的CSV文件包含以下字段：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 字段名
     - 数据类型
     - 说明
   * - ``hash_name``
     - string
     - 文件的哈希名称标识符
   * - ``dataset``
     - string
     - 文件所属的数据集名称
   * - ``file_type``
     - string
     - 文件类型（如pdf、docx、jpg等）
   * - ``source``
     - string
     - 文件的原始来源路径
   * - ``description``
     - string
     - 文件的描述信息（已处理逗号冲突）
   * - ``timestamp``
     - string
     - 文件处理的时间戳

核心功能
--------

数据收集函数
~~~~~~~~~~~~

模块自动执行以下数据收集流程：

1. **JSONL文件扫描**：

   .. code-block:: python

      jsonl_dir = parsed_file_dir / "jsonl"
      for jsonl in jsonl_dir.iterdir():
          with open(jsonl, encoding="utf-8") as f:
              for line in f:
                  data = json.loads(line)
                  file_record.append(data["meta"])

2. **图像描述扫描**：

   .. code-block:: python

      image_desc = parsed_file_dir / "image_desc"
      for image in image_desc.iterdir():
          with open(image, encoding="utf-8") as f:
              for line in f:
                  data = json.loads(line)
                  file_record.append(data["meta"])

数据处理函数
~~~~~~~~~~~~

.. autofunction:: src.tools.file_management.replace_comma

**功能**：将文本中的逗号替换为空格，避免CSV格式冲突。

.. code-block:: python

   def replace_comma(text):
       return text.replace(",", " ")

CSV输出功能
~~~~~~~~~~~~

自动生成CSV文件到 ``data/parsed_file/file_record.csv``：

.. code-block:: python

   with open(parsed_file_dir / "file_record.csv", "w", encoding="utf-8") as f:
       f.write("hash_name,dataset,file_type,source,description,timestamp\n")
       for record in file_record:
           f.write(f"{record['hash_name']},{record['dataset']},{record['file_type']},"
                  f"{record['source']},{replace_comma(record['description'])},"
                  f"{record['timestamp']}\n")

使用方法
--------

直接运行脚本
~~~~~~~~~~~~

.. code-block:: bash

   python src/tools/file_management.py

脚本将自动：

1. 扫描 ``data/parsed_file`` 目录
2. 读取所有JSONL文件
3. 提取文件元数据
4. 生成 ``file_record.csv`` 文件

输出文件
~~~~~~~~

脚本执行后将在 ``data/parsed_file/`` 目录下生成：

* **file_record.csv**：包含所有已解析文件的汇总信息

CSV文件示例：

.. code-block:: csv

   hash_name,dataset,file_type,source,description,timestamp
   abc123def,papers,pdf,/data/input/paper1.pdf,人工智能研究论文,2024-01-15T10:30:00Z
   def456ghi,images,jpg,/data/input/chart1.jpg,数据分析图表,2024-01-15T11:15:00Z

数据验证
--------

模块包含完整的数据验证机制：

* **必需字段检查**：确保每条记录包含所有必需字段
* **数据类型验证**：验证字段值的合理性
* **编码处理**：使用UTF-8编码处理所有文本

.. code-block:: python

   try:
       f.write(f"{record['hash_name']},{record['dataset']},...")
   except KeyError as exc:
       raise KeyError(f"记录缺少必需字段: {record}") from exc

目录结构要求
------------

模块期望以下目录结构存在：

.. code-block::

   data/
   └── parsed_file/
       ├── jsonl/           # 文本解析结果JSONL文件
       │   ├── batch_001.jsonl
       │   ├── batch_002.jsonl
       │   └── ...
       └── image_desc/      # 图像描述JSONL文件
           ├── image_001.jsonl
           ├── image_002.jsonl
           └── ...

依赖项
------

* **标准库**：json, pathlib
* **文件要求**：data/parsed_file目录下的JSONL文件

异常处理
--------

模块处理以下异常情况：

* **文件不存在**：目录或文件不存在时跳过
* **JSON解析错误**：跳过格式错误的JSONL行
* **字段缺失**：明确报告缺少的必需字段
* **编码错误**：使用UTF-8编码处理文件

应用场景
--------

1. **数据汇总**：汇总所有已解析文件的元数据信息
2. **索引构建**：为文件管理系统构建索引
3. **数据分析**：提供文件统计和分析的基础数据
4. **质量检查**：验证文件解析结果的完整性

注意事项
--------

* **运行时机**：应在文件解析流程完成后运行
* **数据一致性**：确保JSONL文件中的meta字段格式统一
* **性能考虑**：对于大量文件，处理时间可能较长
* **输出覆盖**：每次运行会覆盖之前的CSV文件
