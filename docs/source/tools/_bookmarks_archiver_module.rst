.. _bookmarks_archiver_module:

书签网页归档器
~~~~~~~~~~~~~~

本模块实现从 **Chrome 书签文件** 中提取 URL，并使用 **Playwright** 并发抓取网页内容，按原始书签文件夹结构保存为 HTML 文件。  
支持**去重归档**（通过 ``index.json`` 记录已保存 URL）和**并发控制**，适用于构建个人网页知识库或离线阅读库。

功能亮点
--------

- **结构保留**：按 Chrome 书签文件夹层次组织归档目录；
- **智能去重**：每个文件夹下维护 ``index.json``，避免重复抓取同一 URL；
- **并发抓取**：使用 asyncio + 信号量控制并发数（默认 10）；
- **鲁棒性设计**：
  - 超时控制（30 秒）；
  - 异常隔离（单个 URL 失败不影响整体）；
  - 原子性保存 ``index.json``；
- **灵活限制**：支持通过 ``max_pages`` 限制最大抓取数量。

核心组件
--------

.. autoclass:: src.parser.chrome_bookmarks.ChromeBookmarksExtractor
   :members:
   :undoc-members:

.. autoclass:: src.parser.chrome_bookmarks.BookmarksWebArchiver
   :members:
   :undoc-members:

类说明
------

ChromeBookmarksExtractor
~~~~~~~~~~~~~~~~~~~~~~~~

负责解析 Chrome 书签 JSON 文件，递归提取所有 URL。

- **输入**：Chrome 书签文件路径（如 ``~/.config/google-chrome/Default/Bookmarks``）；
- **输出**：按文件夹组织的书签列表，格式：

  .. code-block:: python

     [
       {
         "folder": "技术博客",
         "bookmarks": [
           {"name": "Sphinx 官网", "url": "https://www.sphinx-doc.org/"}
         ]
       }
     ]

BookmarksWebArchiver
~~~~~~~~~~~~~~~~~~~~

核心归档器，整合提取、抓取、保存逻辑。

- **构造参数**：
  - ``bookmark_path``：书签文件路径（必填）；
  - ``save_root_dir``：归档根目录（默认 ``./bookmarks_archive``）；
  - ``max_concurrent``：最大并发抓取数（默认 10）。

- **关键方法**：
  - ``archive(max_pages=None)``：执行归档，返回结果字典；
  - ``_process_bookmark(...)``：处理单个书签（去重 + 抓取 + 保存）；
  - ``_load_index()`` / ``_save_index()``：管理 ``index.json``。

文件命名规则
------------

每个 HTML 文件按以下格式命名：

``<cleaned_domain>_<YYYYMMDD_HHMM>_<url_md5_6>.html``

- ``cleaned_domain``：从 URL 提取的域名，非法字符替换为下划线；
- ``YYYYMMDD_HHMM``：归档时间戳；
- ``url_md5_6``：URL 的 MD5 前 6 位（确保唯一性）。

**示例**：

URL: ``https://example.com/技术/文档.html``
文件名: ``example_com_20251212_1430_a1b2c3.html``

去重机制
--------

每个书签文件夹下维护一个 ``index.json``，格式：

.. code-block:: json

   {
     "https://example.com/page1": "技术博客/example_com_...html",
     "https://example.com/page2": "技术博客/example_com_...html"
   }

归档前检查 URL 是否已存在，若存在则跳过抓取，实现幂等性。

使用示例
--------

命令行运行
~~~~~~~~~~

.. code-block:: bash

   # 默认路径（Linux）
   python src/parser/chrome_bookmarks.py

   # 自定义路径
   python src/parser/chrome_bookmarks.py --bookmark_path "~/Bookmarks" --save_dir "my_archive"

**注意**：当前主函数未暴露命令行参数，需修改 ``if __name__ == "__main__"`` 部分。

API 调用
~~~~~~~~

.. code-block:: python

   from src.parser.chrome_bookmarks import get_bookmarks

   get_bookmarks(
       bookmark_path="~/.config/google-chrome/Default/Bookmarks",
       save_root_dir="data/parsed_file/bookmarks_archive",
       max_pages=50  # 仅抓取前50个书签
   )

各平台书签路径
--------------

- **Linux**: ``~/.config/google-chrome/Default/Bookmarks``
- **Windows**: ``%LOCALAPPDATA%\Google\Chrome\User Data\Default\Bookmarks``
- **macOS**: ``~/Library/Application Support/Google/Chrome/Default/Bookmarks``

**提示**：路径中的 ``~`` 或环境变量需通过 ``os.path.expanduser`` 解析。

异常处理
--------

- **文件未找到**：抛出 ``FileNotFoundError``；
- **JSON 解析失败**：抛出 ``ValueError``；
- **网页抓取失败**：记录错误日志，跳过该 URL，不中断整体流程；
- **文件保存失败**：不影响 ``index.json`` 更新（因仅在成功时记录）。

依赖项
------

- **Python 包**：
  - ``playwright``（需额外安装浏览器：``playwright install chromium``）
  - ``urllib.parse``（标准库）
- **系统依赖**：
  - Chromium 浏览器（由 Playwright 管理）

注意事项
--------

- **权限问题**：确保程序有读取书签文件和写入保存目录的权限；
- **网络超时**：默认 30 秒，对慢速网站可调整 ``page.goto(timeout=...)``；
- **HTML 保存**：仅保存 **DOM 内容** （``domcontentloaded``），不等待图片/视频加载；
- **磁盘空间**：大量网页可能占用较大空间，建议定期清理；
- **法律合规**：请遵守目标网站的 ``robots.txt`` 和使用条款，避免高频抓取。

输出目录结构
------------

::

   bookmarks_archive/
   ├── 技术博客/
   │   ├── example_com_20251212_1430_a1b2c3.html
   │   ├── sphinx_doc_org_20251212_1431_b2c3d4.html
   │   └── index.json
   ├── 新闻/
   │   ├── nytimes_com_20251212_1432_c3d4e5.html
   │   └── index.json
   └── ...

性能调优建议
------------

- **并发数**：根据网络带宽和 CPU 能力调整 ``max_concurrent``；
- **限制数量**：首次运行建议设置 ``max_pages=10`` 测试；
- **去重利用**：中断后重新运行可自动跳过已归档页面；
- **磁盘 I/O**：保存目录建议使用 SSD。
