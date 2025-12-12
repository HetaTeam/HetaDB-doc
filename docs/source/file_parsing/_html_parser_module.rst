.. _html_parser_module:

HTML 批量解析模块
==============================

本模块用于将一批 HTML 文件（本地存储）解析为结构化的统一输出格式（``UnifiedDoc``），支持提取文本、图像、表格、元数据等，并兼容网络 URL 上下文。  
所有中间结果最终合并为标准化的 JSONL 文件输出。

对外接口简洁，仅需调用 ``parse()`` 函数即可完成端到端处理。

功能亮点
--------

- **通用 HTML 解析**：适用于网页快照、爬虫保存的 HTML 文件等场景；
- **多源图像提取**：支持 ``<img>``、``<source>``、``og:image``、CSS ``background-image``、``<video poster>`` 等；
- **噪声过滤**：自动移除 ``<script>``、``<style>``、``<nav>``、``<footer>`` 等非主要内容标签；
- **元数据提取**：自动获取 ``<title>`` 和 ``<meta name="description">`` 或 ``og:description``；
- **图像去重与过滤**：通过可选的图像哈希白名单控制图像保留；
- **统一输出格式**：生成符合 ``UnifiedDoc`` 规范的 JSONL，便于后续检索或入库。

使用示例
--------

.. code-block:: python

   from src.file_parsing.html_parser import parse
   from pathlib import Path

   file_list = [Path("data/raw/page1.html"), Path("data/raw/page2.html")]
   jsonls_dir = Path("data/my_dataset/parsed_file/text_json_out")
   dataset = "web_crawl"
   mapping_json = Path("data/my_dataset/parsed_file/hash_dir/mapping.json")
   url_list = ["https://example.com/page1", "https://example.com/page2"]
   image_hashs = {"a1b2c3...", "d4e5f6..."}  # 可选：仅保留白名单图像

   parse(
       file_list=file_list,
       jsonls_dir=jsonls_dir,
       dataset=dataset,
       mapping_json=mapping_json,
       url_list=url_list,
       image_hashs=image_hashs
   )

函数参考
--------

.. autofunction:: src.file_parsing.html_parser.parse

参数说明
~~~~~~~~

``file_list`` : list[Path]  
   待解析的 HTML 文件路径列表（每个文件为完整 HTML 内容）。

``jsonls_dir`` : Path  
   最终 JSONL 输出目录（通常为 ``text_json_out/``）。

``dataset`` : str  
   数据集名称，用于填充元数据字段。

``mapping_json`` : Path  
   哈希映射文件路径（如 ``hash_dir/mapping.json``），用于还原原始文件名到 ``meta.source``。

``url_list`` : list[str]，可选  
   与 ``file_list`` 一一对应的原始网页 URL，用于：
   - 构造绝对图像 URL（通过 ``urljoin``）
   - 记录上下文（当前未直接写入 meta，但用于图像解析）

``image_hashs`` : set[str]，可选  
   图像 URL 的 SHA256 哈希白名单。若提供，则**仅保留白名单中的图像**；若为空集（默认），则保留所有有效图像。

输出格式
--------

每个 HTML 文件生成一个 ``UnifiedDoc``，结构如下：

.. code-block:: json

   {
     "meta": {
       "source": "original_page.html",
       "hash_name": "a1b2c3d4...html",
       "dataset": "web_crawl",
       "timestamp": "2025-12-12T10:30:45.123456",
       "total_pages": 1,
       "file_type": "html",
       "description": "页面标题,页面描述"
     },
     "json_content": {
       "page_0": [
         { "id": "text_0_0", "type": "text", "text": "正文段落..." },
         { "id": "image_0_0", "type": "image", "url": "a1b2c3...", "caption": "图注" },
         { "id": "table_0_0", "type": "table", "text": "表内文本...", "caption": "表标题" },
         { "id": "merge_text_0", "type": "merge_text", "text": "全文拼接结果..." }
       ]
     }
   }

> **注意**：  
> - 图像的 ``url`` 字段默认为 **SHA256 哈希值**（而非原始 URL），便于去重和安全引用；  
> - ``merge_text_0`` 是该页所有文本（含图注、表文本）的拼接结果，可用于全文检索。

核心解析逻辑
------------

1. **加载 HTML**：读取本地 HTML 文件；
2. **提取元数据**：获取标题和描述；
3. **预提取图像候选**：在清理噪声前，扫描所有可能的图像来源（避免因标签删除丢失图像）；
4. **清理噪声**：移除脚本、样式、导航栏等非主体内容；
5. **遍历 DOM**：递归解析文本、图像、表格等元素，生成结构化内容；
6. **恢复遗漏图像**：将未在 DOM 遍历中处理的原始图像（如 ``og:image``）插入到内容开头；
7. **生成合并文本**：拼接所有文本片段，生成 ``merge_text``；
8. **保存中间文件** → **合并为 JSONL** → **清理临时目录**。

图像处理策略
--------------

- **URL 绝对化**：所有相对 URL 通过 ``urljoin(source_url, relative_url)`` 转为绝对 URL；
- **哈希化存储**：图像 URL 默认通过 SHA256 哈希（``sha256_hash(url)``）作为标识，避免长 URL 和特殊字符问题；
- **白名单过滤**：通过 ``image_hashs`` 参数可实现图像级精细控制（如仅保留 LLM/VLM 已处理的图像）；
- **SVG 过滤**：自动跳过 ``.svg`` 图像（因通常为图标或装饰）。

辅助函数
--------

- ``convert_single()``：解析单个 HTML 字符串，返回结构化结果；
- ``_traverse_dom()``：核心 DOM 遍历器，递归构建页面元素；
- ``_extract_all_image_candidates()``：全面提取图像候选（含社交标签、背景图等）；
- ``_remove_noise_tags()``：移除无关 HTML 标签；
- ``_build_merge_text()``：生成全文拼接文本。

依赖项
------

- **Python 包**：
  - ``beautifulsoup4``（HTML 解析）
  - ``loguru``（日志，间接依赖）
- **内部模块**：
  - ``src.file_parsing.convert_to_unified``（统一数据结构）
  - ``src.file_parsing.unified_output.process_middle_files``（结果合并）

注意事项
--------

- 本模块**不下载**远程资源，仅解析本地 HTML 文件；
- ``url_list`` 长度必须与 ``file_list`` 一致（若提供）；
- 图像白名单 ``image_hashs`` 中的哈希值应为 **URL 的 SHA256**（与 ``sha256_hash()`` 函数一致）；
- 临时目录 ``html_parser_output/`` 在合并完成后**自动删除**；
- 若 HTML 内容为空或解析失败，将生成空文档并跳过，不中断整体流程。