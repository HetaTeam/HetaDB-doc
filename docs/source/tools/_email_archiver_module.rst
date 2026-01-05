.. _email_archiver_module:

邮件读取与归档器
===============

本模块提供从多个邮箱账户拉取邮件、去重归档为结构化 JSON 与附件的能力，支持 **IMAP 协议**、**多账户配置**、**增量同步** 与 **附件保存**。

设计目标是构建可检索、可追溯的企业级邮件知识库，适用于合规存档、客户沟通分析或邮件内容提取场景。

核心特性
--------

- **多邮箱支持**：通过 YAML 配置管理多个账户（Gmail、Outlook、163 等）；
- **环境变量安全**：邮箱账号密码通过 ``.env`` 文件注入，避免硬编码；
- **增量拉取**：仅拉取指定天数（如最近 30 天）内的新邮件；
- **智能去重**：基于 ``账号 + UID`` 建立索引，确保每封邮件仅保存一次；
- **结构化输出**：每封邮件独立目录，包含：
  - ``mail_<UID>.json``：完整元数据与正文；
  - 附件文件（原始格式）；
- **163 邮箱适配**：支持发送 IMAP ID 命令以绕过反爬机制；
- **并发处理**：支持多账户并行拉取（需外部调用 ``process_account``）。

核心组件
--------

.. autoclass:: src.parser.email_archiver.EmailReader
   :members:
   :undoc-members:

.. autoclass:: src.parser.email_archiver.MailToJson
   :members:
   :undoc-members:

配置要求
--------

### 1. 环境变量（.env）

每个邮箱提供商需定义一对环境变量：

.. code-block:: ini

   EMAIL_USER_GMAIL=user@gmail.com
   EMAIL_PASS_GMAIL=app_password_123
   EMAIL_USER_163=user@163.com
   EMAIL_PASS_163=password_456

> **安全提示**：建议使用 **应用专用密码**（而非主密码），尤其对 Gmail/Outlook。

### 2. YAML 配置文件（示例）

.. code-block:: yaml

   - name: "工作邮箱"
     provider: "GMAIL"          # 必填，决定环境变量名
     imap_host: "imap.gmail.com"
     imap_port: 993
     mailbox: "INBOX"
     days: 30                   # 拉取最近30天邮件
     max_mails: 100             # 最多拉取100封（可选）
     save_root: "data/email_archive"

   - name: "163测试"
     provider: "163"
     imap_host: "imap.163.com"
     days: 7
     save_root: "data/email_archive"
     # 可选：163专用IMAP ID
     imap_id_info:
       name: "Outlook"
       vendor: "Microsoft Corporation"

> **注意**：``provider`` 字段值将转为大写并拼接到环境变量名中（如 ``EMAIL_USER_163``）。

类说明
------

### EmailReader

负责连接 IMAP 服务器并拉取邮件。

- **关键方法**：``fetch(account_name)``
- **163 邮箱特殊处理**：若配置中包含 ``imap_id_info``，在登录后发送 ``ID`` 命令，提高连接成功率；
- **邮件对象增强**：为每封 ``MailMessage`` 添加 ``account``、``mailbox``、``save_root`` 属性，便于后续处理。

### MailToJson

负责将邮件对象转换为 JSON 并持久化。

- **去重机制**：
  - 主索引：``save_root/index.json``（记录所有已保存邮件的 ``<account>_<uid>``）；
  - 内存暂存区：避免同一轮次内重复处理；
  - 每轮结束后调用 ``flush_index()`` 持久化新增记录。
- **文件结构**：
  ::
  
     email_archive/
     ├── work_gmail/
     │   ├── work_12345/                 # 账号_uid
     │   │   ├── mail_12345.json
     │   │   ├── invoice.pdf
     │   │   └── report.xlsx
     │   └── index.json                  # 该账户的索引
     └── 163_test/
         ├── user_67890/
         │   ├── mail_67890.json
         │   └── photo.jpg
         └── index.json

- **JSON 结构示例**：

  .. code-block:: json

     {
       "account": "work",
       "uid": "12345",
       "date": "2025-12-12T10:30:45",
       "from": {"name": "张三", "email": "zhangsan@company.com"},
       "to": [{"name": "我", "email": "me@gmail.com"}],
       "subject": "Q4财报",
       "content": {
         "text": "请查收附件...",
         "html": "<div>请查收附件...</div>"
       },
       "attachments": [
         {
           "filename": "Q4_report.pdf",
           "path": "work_12345/Q4_report.pdf",
           "sha256": "a1b2c3...",
           "size": 2048000
         }
       ],
       "meta": {
         "fetched_at": "2025-12-12T14:00:00",
         "source": "imap-tools"
       }
     }

使用示例
--------

### 单账户处理

.. code-block:: python

   from src.parser.email_archiver import process_account
   from src.utils.load_config import load_email_config  # 假设存在

   config = load_email_config("email_config.yaml")
   load_dotenv()  # 加载 .env

   saved = process_account("工作邮箱", config, env_bool=True)
   print(f"保存了 {saved} 封新邮件")

### 多账户并发（需外部实现）

.. code-block:: python

   from concurrent.futures import ThreadPoolExecutor

   with ThreadPoolExecutor() as executor:
       futures = [
           executor.submit(process_account, acc["name"], config, True)
           for acc in config
       ]
       for future in futures:
           future.result()

异常处理
--------

- **配置缺失**：若找不到账户或 ``provider``，抛出 ``ValueError``；
- **认证失败**：IMAP 登录异常会中断单个账户处理，不影响其他账户；
- **邮件解析**：跳过无法解析的邮件（当前未实现，可扩展）；
- **文件写入**：附件或 JSON 保存失败会抛出异常，需上层捕获。

依赖项
------

- **Python 包**：
  - ``imap-tools``（邮件拉取）
  - ``python-dotenv``（环境变量加载）
  - ``pyyaml``（配置解析）
- **系统要求**：
  - 网络可访问目标 IMAP 服务器；
  - 足够的磁盘空间保存附件。

注意事项
--------

- **IMAP 权限**：确保邮箱已启用 IMAP 访问（Gmail 需开启“两步验证 + 应用密码”）；
- **UID 稳定性**：IMAP UID 在邮箱会话内唯一，但**不同服务器可能重复**，因此用 ``账号+UID`` 作为全局唯一键；
- **163 限制**：即使发送 IMAP ID，163 邮箱仍可能限制高频请求，建议降低 ``max_mails``；
- **内存消耗**：大附件会加载到内存，超大邮件需流式处理（当前未实现）；
- **索引原子性**：``index.json`` 通过临时文件写入，避免写入中断导致损坏。

合规与安全
-----------

- **数据隐私**：邮件内容可能包含敏感信息，确保存储目录权限受控；
- **遵守条款**：请遵守邮箱服务商的使用条款，避免触发反爬封禁；
- **审计日志**：建议记录 ``meta.fetched_at`` 用于操作审计。
