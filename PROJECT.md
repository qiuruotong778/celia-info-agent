CustomerAI

1. 项目目标

创建一个只运行在本地 Mac 上的客户沟通整理系统。

系统用于自动或半自动收集来自以下渠道的客户沟通内容：

* Email
* WhatsApp
* WeChat
* 后续可扩展其他渠道

系统需要使用本地 AI 自动识别：

* 客户公司
* 联系人
* 项目
* 沟通主题
* 客户需求
* 我方承诺
* 待办事项
* Deadline
* 关键决定
* 是否需要跟进

最终形成：

* 客户档案
* 项目档案
* 时间线
* Open Items
* Decisions
* 每日 Brief
* 本地可搜索 Dashboard

⸻

2. 核心原则

2.1 Local First

默认所有客户数据只保存在本机。

禁止默认将以下信息上传至外部 AI 服务：

* 客户聊天记录
* 邮件正文
* 附件
* 项目信息
* 客户名称
* 联系人信息

AI 默认使用：

Ollama + Local LLM

第一版不使用 OpenAI API。

但代码架构需要预留 AI Provider abstraction，使未来可以切换：

AI_PROVIDER=ollama

未来可以扩展：

AI_PROVIDER=openai

⸻

3. 用户背景

用户不是程序员。

因此整个项目需要：

* 安装简单
* 启动简单
* 出错时提示清楚
* 尽量不需要 Terminal 操作
* 配置集中
* README 必须提供一步一步操作说明
* 所有复杂操作尽可能提供脚本

不要假设用户了解：

* Python
* API
* SQL
* Docker
* Terminal
* JSON
* 环境变量

如必须使用这些概念，请在 README 中用简单语言解释。

⸻

4. 第一版技术栈

优先使用简单稳定的方案。

Backend:

Python 3
FastAPI

Database:

SQLite

ORM:

SQLModel

Local AI:

Ollama

Frontend:

优先：

FastAPI + Jinja2

或者：

简单 HTML / CSS / JavaScript

第一版不要使用复杂前端框架。

不要使用：

React
Next.js
Docker
Kubernetes
PostgreSQL
Redis

除非后续确实有必要。

⸻

5. 第一版文件结构

建议：

customer-ai/
├── PROJECT.md
├── README.md
├── requirements.txt
├── .gitignore
├── .env.example
│
├── app/
│   ├── main.py
│   │
│   ├── config.py
│   ├── database.py
│   │
│   ├── models/
│   │   ├── message.py
│   │   ├── client.py
│   │   ├── contact.py
│   │   ├── project.py
│   │   ├── task.py
│   │   └── decision.py
│   │
│   ├── services/
│   │   ├── ai_service.py
│   │   ├── ingestion_service.py
│   │   ├── classification_service.py
│   │   └── brief_service.py
│   │
│   ├── importers/
│   │   ├── email_importer.py
│   │   ├── whatsapp_importer.py
│   │   └── clipboard_importer.py
│   │
│   ├── templates/
│   └── static/
│
├── data/
│   ├── inbox/
│   │   ├── email/
│   │   ├── whatsapp/
│   │   └── manual/
│   │
│   ├── archive/
│   ├── exports/
│   └── customer_ai.db
│
├── scripts/
│   ├── setup_mac.sh
│   ├── start.sh
│   ├── stop.sh
│   ├── run_ingestion.sh
│   └── install_scheduler.sh
│
└── tests/

data/ 必须加入 .gitignore。

任何客户真实数据不得提交至 GitHub。

⸻

6. 数据模型

Message

每条原始沟通记录保存：

id
source
external_id
timestamp
sender
receiver
company
contact
project
subject
raw_content
summary
created_at
processed_at

source 包括：

email
whatsapp
wechat
manual

⸻

Client

id
name
aliases
notes
created_at
updated_at

示例：

CellCarta

aliases 可以包括：

CC
Cell Carta

⸻

Contact

id
name
client_id
email
phone
role
notes

⸻

Project

id
name
client_id
status
summary
created_at
updated_at

示例：

CCR8
HER2 Demo
Leica IMS

⸻

Task

id
client_id
project_id
source_message_id
title
description
status
priority
deadline
owner
created_at
updated_at
completed_at

status：

open
waiting_customer
waiting_internal
done
cancelled

⸻

Decision

保存已经确定的重要事实。

id
client_id
project_id
source_message_id
title
content
decision_date
created_at

例如：

Title:
CCR8 software version
Content:
Production submission version is V1.0.0.1.

⸻

7. AI 信息提取

每条消息导入后，调用 Ollama。

AI 返回严格结构化 JSON。

格式：

{
  "client": "",
  "contact": "",
  "project": "",
  "topic": "",
  "summary": "",
  "customer_requests": [],
  "our_commitments": [],
  "tasks": [],
  "decisions": [],
  "deadline": null,
  "follow_up_required": false,
  "priority": "normal"
}

priority：

low
normal
high
urgent

⸻

8. AI Prompt 原则

AI 必须：

1. 不编造信息。
2. 不确定的信息设为 null。
3. 客户提出的问题不能自动视为已经完成。
4. “I’ll check internally” 应识别为待办。
5. “We’ll send this later” 应识别为待办。
6. 客户明确确认的事项可记录为 Decision。
7. 客户要求提供资料、文件、确认或答复，应识别为 Task。
8. 如果没有明确 Deadline，不要猜日期。
9. 保留原始消息，AI 摘要不能替代原始记录。
10. 每个 AI 结果必须能够追溯到 source_message_id。

⸻

9. WhatsApp 第一版

第一版不自动读取 WhatsApp 数据库。

用户手动使用：

Export Chat

将 .txt 文件保存至：

data/inbox/whatsapp/

系统自动检测新文件。

需要支持常见 WhatsApp 导出格式。

解析：

date
time
sender
message

同一条消息不得重复导入。

处理完成后，将文件移动到：

data/archive/whatsapp/

⸻

10. 微信第一版

第一版禁止：

* 破解微信数据库
* Hook 微信
* 解密微信数据库
* 修改微信客户端

采用 Manual Inbox。

用户将聊天文字复制进：

data/inbox/manual/

支持 .txt 和 .md。

也提供 Dashboard 输入框：

Paste Message

用户复制微信聊天后粘贴。

字段：

Source: WeChat
Client:
Contact:
Project:
Content:

其中 Client / Contact / Project 可以为空。

AI 自动判断。

⸻

11. Email

Email 分两阶段。

Phase 1

支持 .eml 文件导入。

用户将 .eml 放入：

data/inbox/email/

系统读取：

From
To
Date
Subject
Body

附件第一版只记录：

attachment filename

暂时不解析附件内容。

⸻

Phase 2

后续增加 Gmail 自动同步。

必须使用正常官方授权方式。

禁止：

* 保存 Gmail 密码
* 模拟浏览器登录
* 抓取 Gmail 页面

⸻

12. Dashboard

本地地址：

http://127.0.0.1:8765

只绑定：

127.0.0.1

不要默认暴露：

0.0.0.0

⸻

Dashboard 首页显示：

Today

New Messages
Open Tasks
Waiting Customer
Waiting Internal
Overdue

⸻

Clients

客户列表。

例如：

CellCarta
Leica
Distributor A
Hospital B

点击客户进入客户主页。

⸻

13. Client Page

客户页面需要显示：

Client Name
Overview
Contacts
Projects
Open Items
Recent Messages
Decisions
Timeline

⸻

14. Project Page

例如：

CellCarta / CCR8

展示：

Project Summary
Open Tasks
Waiting Customer
Waiting Internal
Recent Communications
Key Decisions
Timeline

⸻

15. Search

第一版支持普通关键词搜索。

可搜索：

客户
联系人
项目
消息正文
摘要
Task
Decision

例如：

CCR8 API

返回所有相关记录。

第一版不需要 Vector Database。

⸻

16. Daily Brief

每天生成：

data/exports/YYYY-MM-DD-brief.md

格式：

# Customer Brief
## Need Attention
### Client / Project
- Task
- Deadline
- Last communication
## Waiting for Customer
...
## Waiting Internally
...
## Completed Today
...
## Possible Forgotten Items
...

Possible Forgotten Items 规则：

如果：

Task = open

并且超过：

7 days

没有新沟通，则提示。

天数以后可以配置。

⸻

17. Scheduler

Mac 使用：

launchd

不要依赖第三方 scheduler。

第一版：

每 30 分钟：

scan inbox

每天：

17:30

生成 Daily Brief。

Scheduler 必须提供安装脚本：

scripts/install_scheduler.sh

README 清楚说明：

如何安装
如何暂停
如何删除

⸻

18. Ollama

安装后自动检测：

http://localhost:11434

如果 Ollama 没启动：

Dashboard 显示：

Local AI is not running.
Please open Ollama.

不要导致整个应用崩溃。

⸻

模型名称放在 .env：

OLLAMA_MODEL=qwen3

不要把模型名写死在代码里。

⸻

19. AI Provider Interface

实现统一接口：

class AIProvider:
    def analyze_message(self, content):
        pass
    def summarize_project(self, messages):
        pass
    def generate_daily_brief(self, data):
        pass

第一版实现：

OllamaProvider

未来可以增加：

OpenAIProvider

业务代码不得直接依赖 Ollama。

⸻

20. Privacy

必须做到：

data/
.env
*.db

进入 .gitignore。

README 明确提醒：

Do not commit customer data to GitHub.

日志中避免保存完整客户消息。

日志只记录：

message id
processing status
error
timestamp

⸻

21. Backup

Dashboard 提供：

Export Backup

生成：

customer-ai-backup-YYYYMMDD.zip

包含：

SQLite database
Markdown exports
configuration without secrets

第一版不需要自动上传云盘。

⸻

22. Manual Correction

AI 会犯错。

因此 Dashboard 必须允许用户修改：

Client
Contact
Project
Task
Decision
Deadline
Status

所有 AI 结果都必须可编辑。

⸻

23. Merge / Alias

客户可能有多个名称。

例如：

CellCarta
Cell Carta
CC

需要支持 alias。

如果 AI 认为两个客户可能相同：

不要自动合并。

显示：

Possible duplicate client

由用户确认。

⸻

24. 第一阶段开发范围

请严格按照以下顺序完成。

Milestone 1

建立基础项目：

* FastAPI
* SQLite
* SQLModel
* Folder structure
* Configuration
* start.sh

确保：

./scripts/start.sh

可以启动。

⸻

Milestone 2

建立：

Client
Contact
Project
Message
Task
Decision

数据表。

⸻

Milestone 3

实现 Manual Inbox。

支持：

.txt
.md

文件导入。

⸻

Milestone 4

接入 Ollama。

实现：

analyze_message()

并写入数据库。

⸻

Milestone 5

实现 Dashboard：

Home
Clients
Client Detail
Projects
Tasks
Messages

⸻

Milestone 6

WhatsApp Export Parser。

⸻

Milestone 7

.eml Email Importer。

⸻

Milestone 8

Daily Brief。

⸻

Milestone 9

launchd Scheduler。

⸻

Milestone 10

Backup。

⸻

25. 暂时不要开发

第一版禁止加入：

微信数据库破解
WhatsApp Web 自动抓取
浏览器自动化
邮件自动发送
WhatsApp 自动发送
微信自动发送
自动回复客户
Vector Database
RAG Framework
Docker
Cloud Database
User Login System
Multi-user
Mobile App

保持系统简单。

⸻

26. 开发方式

Codex 开发时：

每完成一个 Milestone：

1. 运行测试。
2. 实际启动程序。
3. 检查错误。
4. 修复后再进入下一阶段。

不要一次性生成整个项目而不测试。

⸻

27. Test Data

只能使用虚构测试数据。

例如：

Company:
Example Pharma
Contact:
John Smith
Project:
Biomarker Study

禁止将真实客户信息写入：

tests/
README
GitHub
example files

⸻

28. README

README 必须面向完全没有编程经验的人。

至少包括：

First Time Setup

Step 1
Install Python
Step 2
Install Ollama
Step 3
Download local AI model
Step 4
Run setup script
Step 5
Start CustomerAI

Every Day

说明：

如何启动
如何停止
如何导入微信
如何导入WhatsApp
如何导入邮件
如何查看Daily Brief

Troubleshooting

包括：

Ollama not running
Port already in use
Python not installed
AI model not installed
Database error
Inbox file could not be parsed

⸻

29. 用户体验

用户应该最终只需要：

启动：

CustomerAI

然后浏览器打开：

http://127.0.0.1:8765

平时：

微信 → Copy/Paste
WhatsApp → Export Chat
Email → Export .eml

其余整理工作自动完成。

⸻

30. 最终目标

CustomerAI 应成为一个本地客户记忆系统。

用户应该可以询问或查看：

CellCarta 最近还有哪些事情没处理？
CCR8 项目现在有哪些 Open Items？
Dr. X 最近提出过哪些要求？
哪些客户超过 7 天没有跟进？
这个需求最早是谁什么时候提出的？
某项决定是在哪条消息确认的？

所有答案必须可以追溯到原始消息。

AI 只负责：

Read
Extract
Organize
Summarize
Remind

第一版不负责：

Send
Delete
Reply
Modify external platforms
