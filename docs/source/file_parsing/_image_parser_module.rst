.. _image_parser_module:

图像描述生成模块
============================

本模块利用视觉语言模型（VLM）为图像生成结构化描述（标题 + 内容），支持两类输入：

1. **原始图像文件** （如 ``.jpg``, ``.png``）；
2. **从 JSONL 文档中提取的图像** （如 PDF/HTML 解析产物）。

- 所有结果统一输出为符合 ``UnifiedDoc`` 规范的 JSON 文件，并最终合并为滚动式 JSONL。

- 核心目标：为每张图像生成高质量、可检索的语义描述，用于知识库构建或图谱关联。

功能特点
--------

- **异步批量处理**：基于 ``asyncio`` 和 ``tqdm_asyncio``，高效并发调用 VLM；
- **上下文感知**：为文档内图像提供参考文本（如页内文字、图注、元描述）；
- **双源输入支持**：
  - 直接处理原始图像文件；
  - 从已有的 JSONL 文档中提取图像元素并补充描述；
- **去重机制**：通过图像哈希（或 URL）避免重复处理；
- **结构化输出**：每张图像独立生成 JSON，包含元数据、标题、描述；
- **容错与重试**：自动跳过无效图像，VLM 调用失败时最多重试 3 次。

使用示例
--------

.. code-block:: python

   from src.file_parsing.image_parser import create_description
   from pathlib import Path
   import asyncio

   # 假设已初始化 VLM 客户端（需实现 callable 接口）
   async def vlm_client(prompt: str, base64_img: str, mime: str):
       # 实际调用 OpenAI/Gemini/Qwen-VL 等
       return '{"caption": "...", "desc": "..."}'

   raw_images = [Path("data/raw/img1.jpg"), Path("data/raw/img2.png")]
   jsonl_images_dir = Path("data/my_dataset/parsed_file/image_dir")
   jsonl_path_list = Path("data/my_dataset/parsed_file/text_json_out")
   image_desc_dir = Path("data/my_dataset/parsed_file/image_desc_out")
   dataset = "multimodal_corpus"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")

   asyncio.run(
       create_description(
           file_list=raw_images,
           jsonl_images_dir=jsonl_images_dir,
           jsonl_path_list=jsonl_path_list,
           image_desc_dir=image_desc_dir,
           dataset=dataset,
           mapping_json=mapping_json,
           vlm=vlm_client,
       )
   )

函数参考
--------

.. autofunction:: src.file_parsing.image_parser.create_description

参数说明
~~~~~~~~

``file_list`` : list[Path]  
   原始图像文件路径列表（直接来自 ``hash_dir/``）。

``jsonl_images_dir`` : Path  
   JSONL 中引用的图像文件所在目录（通常是 ``image_dir/``）。

``jsonl_path_list`` : Path  
   包含图像引用的 JSONL 文件所在目录（通常是 ``text_json_out/``），模块会自动查找其下所有 ``*.jsonl`` 文件。

``image_desc_dir`` : Path  
   最终图像描述 JSONL 输出目录。

``dataset`` : str  
   数据集名称，用于填充元数据。

``mapping_json`` : Path  
   哈希映射文件路径，用于还原原始文件名。

``vlm`` : Callable  
   异步 VLM 客户端函数，需满足签名：
   .. code-block:: python

      async def vlm(prompt: str, base64_image: str, mime_type: str) -> str:
          ...

   返回值应为包含 ``{"caption": "...", "desc": "..."}`` 的 JSON 字符串（或可解析子串）。

VLM Prompt 设计
---------------

系统使用以下提示词调用模型：

.. code-block::

   请给这张图片提供说明，识别图中关键标识性元素，并推测图片标题；
   [这张图片的caption是xxx]（如有）
   可参考文字内容：yyy，但要仔细甄别出与图片相关的内容
   要求语言简洁凝练，不要描述画面布局；
   严格按照下列JSON格式输出，不要其他任何解释：{"caption": "图片标题", "desc": "图片内容描述"}

> **说明**：  
> - ``xxx`` 来自图像的 ``alt``/``title``/``caption`` 字段；  
> - ``yyy`` 来自同页或邻近页的文本内容（``merge_text``）及文档元描述。

输出格式
--------

每张图像生成一个 JSON 文件（临时），结构为 ``UnifiedDoc``：

.. code-block:: json

   {
     "meta": {
       "source": "original_file.pdf",
       "hash_name": "a1b2c3d4...jpg",
       "dataset": "multimodal_corpus",
       "timestamp": "2025-12-12T10:30:45.123456",
       "total_pages": 1,
       "file_type": "image",
       "description": "城市夜景: 上海陆家嘴金融区夜晚航拍..."
     },
     "json_content": {
       "page_0": [
         {
           "id": "image_0_0",
           "type": "image",
           "source": "original_file.pdf",
           "hash_name": "a1b2c3d4...jpg",
           "caption": "城市夜景",
           "desc": "上海陆家嘴金融区夜晚航拍..."
         }
       ]
     }
   }

所有中间 JSON 文件最终通过 ``process_middle_files`` 合并为滚动式 JSONL（默认 3GB/文件）。

图像处理流程
------------

1. **加载哈希映射**：还原原始文件名；
2. **处理 JSONL 中的图像**：
   - 遍历所有 JSONL 文件，提取 ``type=="image"`` 的元素；
   - 通过 ``url`` 字段定位本地图像文件；
   - 构造参考文本（图注 + 页内文字 + 文档描述）；
   - 调用 VLM 生成描述；
3. **处理原始图像文件**：
   - 直接调用 VLM，无参考文本；
4. **去重**：通过 ``image_parsed_set`` 确保每张图像仅处理一次；
5. **输出 & 合并**：生成中间 JSON → 合并为 JSONL → 清理临时目录。

工具函数
--------

.. autofunction:: src.file_parsing.image_parser.get_image_desc_async

.. autofunction:: src.file_parsing.image_parser.is_valid_image

.. autofunction:: src.file_parsing.image_parser.get_image_mime

依赖项
------

- **Python 包**：

  - ``Pillow`` （图像验证）
  - ``openai`` （示例 VLM 客户端，实际可替换）
  - ``tqdm`` （进度条）
  - ``loguru`` / ``logging`` （日志）
  - ``pyyaml`` （可能用于配置加载）
  
- **内部模块**：

  - ``src.file_parsing.convert_to_unified`` （统一数据结构）
  - ``src.file_parsing.unified_output.process_middle_files`` （结果合并）

注意事项
--------

- **VLM 客户端需自行实现**：模块不绑定具体模型，只需提供兼容的异步函数；
- **图像路径依赖**：JSONL 中的 ``url`` 必须能对应到 ``jsonl_images_dir`` 下的文件；
- **临时目录**：``image_parser_temp_output/`` 在合并后自动删除；
- **无效图像跳过**：损坏或非图像文件会被静默跳过；
- **输出粒度**：每张图像独立输出，便于后续按需加载或嵌入。
