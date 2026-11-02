# Mini-RAG 知识库问答

Mini-RAG 是一个轻量级本地知识库问答应用。用户可以上传文本资料，系统会对文档进行分块并使用 TF-IDF 建立检索索引；提问时，系统先检索相关资料，再将检索结果作为上下文交给 DeepSeek 模型，并通过 SSE 实时返回回答。

前端使用原生 HTML、CSS 和 JavaScript，后端使用 Flask，数据以 JSON 文件形式保存在本地，适合用于个人知识库、产品资料问答和 RAG 流程演示。

## 功能特性

- 支持上传 `.txt`、`.md`、`.json`、`.csv` 等文本类文件。
- 按字符滑动窗口切分文档，支持配置分块大小和重叠长度。
- 使用字符级 TF-IDF 向量进行余弦相似度检索。
- 将检索到的文档片段注入提示词，约束模型基于资料回答。
- 通过 SSE 流式展示模型回答。
- 支持创建、切换和删除多个会话，自动保存聊天记录。
- 支持查看和删除知识库文档。

## 技术栈

- Python 3.10+
- Flask
- OpenAI Python SDK（兼容 DeepSeek API）
- scikit-learn
- NumPy
- python-dotenv
- 原生 HTML、CSS、JavaScript

## 工作原理

```text
上传文档
   │
   ▼
文本分块 ──► TF-IDF 向量化 ──► data/docstore.json
                                      │
用户提问 ──► 问题向量化 ──► 相似度检索 ┘
                                      │
                                      ▼
                         拼接上下文并调用 DeepSeek
                                      │
                                      ▼
                              SSE 流式返回回答
                                      │
                                      ▼
                          保存到 data/sessions/*.json
```

## 项目结构

```text
rag/
├── app.py                 # Flask 应用、页面和 API 路由
├── chat.py                # 命令行对话入口
├── config.py              # 环境变量和运行参数
├── llm.py                 # DeepSeek 客户端与流式调用
├── rag.py                 # 文档分块、检索和向量索引
├── storage.py             # 文档与会话的 JSON 持久化
├── requirements.txt       # Python 依赖
├── static/
│   ├── index.html         # Web 页面
│   ├── css/style.css      # 页面样式
│   └── js/app.js          # 会话、上传和流式聊天逻辑
├── data/
│   ├── docstore.json      # 文档、分块和向量数据
│   ├── vectorizer.pkl     # TF-IDF 向量器
│   └── sessions/          # 会话记录
└── 测试文档/               # 示例产品手册
```

## 快速开始

### 1. 创建虚拟环境并安装依赖

Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

macOS / Linux：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 2. 配置环境变量

在项目根目录创建或编辑 `.env`：

```dotenv
DEEPSEEK_API_KEY=填写你的_API_Key
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-v4-flash

CHUNK_SIZE=400
CHUNK_OVERLAP=100
RETRIEVAL_TOP_K=4
```

`DEEPSEEK_API_KEY` 为必需项，其余参数均有代码内默认值。请勿将包含真实 API Key 的 `.env` 提交到公开仓库。

### 3. 启动 Web 应用

```powershell
python app.py
```

浏览器访问：

```text
http://127.0.0.1:8000
```

启动后，在左侧知识库中上传文本文件，新建或选择一个会话，然后输入问题即可。

### 4. 可选：使用命令行对话

命令行模式不包含知识库检索，仅用于直接测试 DeepSeek 连接：

```powershell
python chat.py
```

输入 `exit` 或 `quit` 退出。

## 配置说明

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `DEEPSEEK_API_KEY` | 无 | DeepSeek API 密钥，必需 |
| `DEEPSEEK_BASE_URL` | `https://api.deepseek.com` | OpenAI 兼容接口地址 |
| `DEEPSEEK_MODEL` | `deepseek-v4-flash` | 调用使用的模型名称 |
| `CHUNK_SIZE` | `400` | 每个文本块的字符数 |
| `CHUNK_OVERLAP` | `100` | 相邻文本块的重叠字符数 |
| `RETRIEVAL_TOP_K` | `4` | 每次检索返回的文本块数量 |

`CHUNK_OVERLAP` 应小于 `CHUNK_SIZE`，否则分块可能无法正常结束。调整分块参数后，建议重新上传文档以重建索引。

## API 说明

所有接口以 `/api` 为前缀，默认返回 JSON；聊天接口使用 `text/event-stream`。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api/documents` | 获取已入库文档列表 |
| `POST` | `/api/documents/upload` | 上传文档，使用 `multipart/form-data`，字段名为 `file` |
| `DELETE` | `/api/documents/<filename>` | 按文件名删除文档及其文本块 |
| `GET` | `/api/sessions` | 获取会话列表 |
| `POST` | `/api/sessions` | 创建会话，可传 JSON 字段 `title` |
| `GET` | `/api/sessions/<session_id>` | 获取会话详情和消息记录 |
| `DELETE` | `/api/sessions/<session_id>` | 删除会话 |
| `POST` | `/api/chat/stream` | 基于知识库检索进行流式问答 |

### 流式问答请求

```json
{
  "message": "这款产品如何充电？",
  "session_id": "已创建的会话 ID"
}
```

SSE 响应包含以下事件：

```text
data: {"type":"meta","retrieved_count":4}
data: {"type":"content","text":"根据知识库资料，..."}
data: {"type":"error","message":"错误信息"}
data: [DONE]
```

- `meta`：本次检索到的文本块数量。
- `content`：模型生成的文本增量。
- `error`：调用模型或检索过程中出现的错误。
- `[DONE]`：流式响应结束。

省略 `session_id` 时，服务端会自动创建会话，但当前响应不会返回新会话 ID。调用 API 时建议先创建会话并显式传入 `session_id`。

## 数据持久化

- 文档、文本块及向量保存在 `data/docstore.json`。
- TF-IDF 向量器保存在 `data/vectorizer.pkl`。
- 每个会话单独保存在 `data/sessions/<session_id>.json`。
- 删除文档或会话后，对应 JSON 数据会同步更新。

当前存储方案没有并发写入锁，适合单用户本地使用，不建议直接用于多人并发生产环境。

## 使用限制

- 文档上传仅按 UTF-8 文本读取，不支持 PDF、Word、图片或 OCR；非 UTF-8 内容可能被替换字符破坏。
- 当前检索基于 TF-IDF 词法相似度，不是基于语义向量的嵌入检索，同义表达召回能力有限。
- 每次新增文档都会重建全量向量器并重新编码所有文本块，文档较多时上传耗时和内存占用会增加。
- Web 应用未实现用户登录、权限控制、上传大小限制或接口鉴权。
- `app.py` 默认启用 Flask 调试模式并监听 `0.0.0.0`，只适合本地开发，不应直接暴露到公网。

## 常见问题

### 提示 `DEEPSEEK_API_KEY 未配置`

确认项目根目录存在 `.env`，且其中已填写有效的 `DEEPSEEK_API_KEY`。修改后需要重启应用。

### 模型回复“知识库中暂无相关内容”

先确认文档上传成功，并尝试使用与原文更接近的词语提问。TF-IDF 对同义词和改写句式的匹配能力有限。

### 上传后回答仍然使用旧内容

检查 `data/docstore.json` 是否已更新。若手动修改过 `data` 目录，应停止服务、恢复一致的数据文件后重新启动。

### PowerShell 无法执行虚拟环境激活脚本

可在当前 PowerShell 会话中执行：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## 后续改进方向

- 使用语义嵌入模型和专用向量数据库替代 TF-IDF。
- 增加 PDF、Word、网页等格式的解析能力。
- 为文档上传增加大小、类型和数量限制。
- 增加用户认证、访问控制和并发写入保护。
- 将文档与会话迁移到 SQLite 或服务端数据库。
