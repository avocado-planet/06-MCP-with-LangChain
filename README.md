# MCP（Model Context Protocol）解説ガイド

## MCPとは

**Model Context Protocol (MCP)** は、Anthropic が開発した **オープンプロトコル（通信規格）** で、LLM アプリケーションが外部のツールやデータソースに標準化された方法で接続するための仕組みです。2025年にLinux Foundation傘下のAgentic AI Foundationに寄贈され、OpenAI・Google・Microsoft・AWS なども支持しています。

### なぜ MCP が必要か

MCP がない場合、LLM アプリから各サービスに接続するために個別のカスタムコードが必要です。サービスごとに認証方式、API形式、エラー処理が異なるため、統合コストが膨大になります。

MCP があると、各サービスは「MCPサーバ」として1度だけプロトコルを実装すれば、どんな「MCPクライアント」からも接続できます。USB-C のような統一コネクタです。

```
MCP なし:                        MCP あり:
LLM ──独自コード→ GitHub API     LLM ──MCP→ GitHub MCPサーバ
LLM ──独自コード→ Slack API      LLM ──MCP→ Slack MCPサーバ
LLM ──独自コード→ DB             LLM ──MCP→ DB MCPサーバ
(それぞれ別の実装が必要)          (全て同じプロトコル)
```

---

## アーキテクチャ

```
┌────────────────┐                    ┌────────────────┐
│   MCP Client   │   JSON-RPC 2.0    │   MCP Server   │
│                │◄──────────────────►│                │
│  (LangChain    │                    │  (FastMCP 等)  │
│   エージェント)  │                    │                │
└────────────────┘                    └────────────────┘
        │                                     │
        │  tools/list                         │ @mcp.tool()
        │  → サーバが持つツール一覧を取得       │ で関数を公開
        │                                     │
        │  tools/call                         │
        │  → ツールを引数付きで実行            │
        │  ← 結果を返す                        │
```

MCPは JSON-RPC 2.0 ベースのプロトコルで、主に2つのメソッドがあります:

- `tools/list` — サーバが公開しているツールの一覧（名前・説明・引数スキーマ）を取得
- `tools/call` — 指定したツールを引数付きで実行し、結果を取得

---

## 通信方式（Transport）

### stdio（Standard I/O）

クライアントがサーバを**サブプロセスとして起動**し、標準入出力（stdin/stdout）で通信します。

```python
client = MultiServerMCPClient({
    "math": {
        "transport": "stdio",
        "command": "python",
        "args": ["/path/to/math_server.py"],
    },
})
```

- サーバはクライアントと同じマシン上で動作
- プロセスのライフサイクルはクライアントが管理
- セットアップが最もシンプル
- 1対1接続（クライアント1つにつきサーバプロセス1つ）

### HTTP（streamable-http）

サーバが**独立したHTTPサーバ**として動作し、HTTPリクエストで通信します。

```python
client = MultiServerMCPClient({
    "weather": {
        "transport": "http",
        "url": "http://localhost:8000/mcp",
    },
})
```

- サーバはリモートマシンでもOK
- 複数クライアントが同時接続可能
- 認証ヘッダーを付けられる（`headers` フィールド）
- 本番環境向け

### 使い分け

| 項目 | stdio | HTTP |
|---|---|---|
| 接続先 | ローカルのみ | ローカル / リモート |
| 起動管理 | クライアントが起動 | サーバを別途起動 |
| 複数クライアント | 不可 | 可能 |
| 認証 | 不要 | ヘッダーで対応可 |
| 用途 | 開発・テスト・ローカルツール | 本番・共有サービス |

---

## MCPサーバの作成（FastMCP）

`FastMCP` は Python の関数を MCP ツールとして公開するためのライブラリです。

### 最小のサーバ

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")  # サーバ名

@mcp.tool()
def add(a: int, b: int) -> int:
    """2つの数を足し算します"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """2つの数を掛け算します"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="sse")
```

ポイント:
- `@mcp.tool()` デコレータで関数をツールとして公開
- **docstring がツールの説明**になる（LLM がいつ使うかを判断する材料）
- 型ヒント（`a: int, b: int`）が引数スキーマになる
- `mcp.run(transport="sse")` で SSE サーバとして起動（Colab では stdio の代わりに SSE を使う）

### HTTP サーバの場合

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", port=8000)
```

起動後、`http://localhost:8000/mcp` で接続できます。

> **Note:** Colab では `streamable-http` の代わりに `sse` を使うと安定します（`fileno()` の制約を回避）。

---

## LangChain からの接続

### langchain-mcp-adapters

MCP ツールを LangChain のツールに変換するアダプタライブラリです。

```bash
pip install langchain-mcp-adapters langchain-openai
```

### 方法1: MultiServerMCPClient（SSE、Colab対応）

```python
import subprocess, time
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

# サーバをバックグラウンドで SSE 起動（デフォルトポート8000）
proc = subprocess.Popen(["python", "/path/to/math_server.py"],
                        stdout=subprocess.PIPE, stderr=subprocess.PIPE)
time.sleep(2)

model = ChatOpenAI(model="gpt-4.1-mini")

# async with は使えない（v0.1.0以降）
client = MultiServerMCPClient({
    "math": {
        "url": "http://localhost:8000/sse",
        "transport": "sse",
    }
})
tools = await client.get_tools()  # await が必要

agent = create_agent(model, tools)
result = await agent.ainvoke(
    {"messages": [{"role": "user", "content": "(3 + 5) × 12 を計算して"}]}
)
```

> **Note:** Colab では `stdio_client` が `fileno()` エラーになるため、SSE 方式を使います。

> **Note:** `langchain-mcp-adapters 0.1.0` 以降、`MultiServerMCPClient` はコンテキストマネージャ（`async with`）として使えません。`client = MultiServerMCPClient(...)` の後に `await client.get_tools()` を呼びます。

### 方法2: MultiServerMCPClient（複数サーバ）

複数サーバに接続する場合、**ポートを変えたいサーバは `uvicorn` で起動**します。`FastMCP.run()` は `port` 引数を受け付けないためです。

```python
# ポート8001で起動したいサーバの場合
import uvicorn
uvicorn.run(mcp.sse_app(), host="0.0.0.0", port=8001)
```

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

client = MultiServerMCPClient({
    "math": {
        "url": "http://localhost:8000/sse",  # mcp.run(transport="sse") のデフォルト
        "transport": "sse",
    },
    "text": {
        "url": "http://localhost:8001/sse",  # uvicorn で port=8001 指定
        "transport": "sse",
    },
})
tools = await client.get_tools()
agent = create_agent(model, tools)
```

- 辞書のキー（`"math"`, `"weather"`）はサーバの識別名
- `get_tools()` で全サーバからツールをまとめて取得
- デフォルトでステートレス（各ツール呼び出しで新しいセッションを作成）

### MCP ツール + 通常ツールの混在

MCP から取得したツールと `@tool` デコレータのツールは同じリストに入れられます。

```python
from langchain_core.tools import tool

@tool
def get_time() -> str:
    """現在時刻を返します"""
    from datetime import datetime
    return datetime.now().isoformat()

# MCP ツール + 通常ツールを合体
all_tools = [get_time] + mcp_tools
agent = create_agent(model, all_tools)
```

### LangGraph の StateGraph で使う

`create_agent` の代わりに `StateGraph` で使うことも可能です。

```python
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4.1-mini")
model_with_tools = model.bind_tools(tools)

def call_model(state: MessagesState):
    return {"messages": model_with_tools.invoke(state["messages"])}

builder = StateGraph(MessagesState)
builder.add_node("model", call_model)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "model")
builder.add_conditional_edges("model", tools_condition)
builder.add_edge("tools", "model")

graph = builder.compile()
```

---

## HTTP 接続で認証を付ける

```python
client = MultiServerMCPClient({
    "my_service": {
        "transport": "http",
        "url": "https://api.example.com/mcp",
        "headers": {
            "Authorization": "Bearer YOUR_TOKEN",
            "X-Custom-Header": "value",
        },
    },
})
```

---

## 公開されている MCP サーバ

Anthropic と コミュニティが多数の MCP サーバを公開しています:

- **Google Drive** — ドキュメントの検索・読み取り
- **Slack** — メッセージの送受信
- **GitHub** — リポジトリ操作、Issue 管理
- **PostgreSQL** — データベースクエリ
- **Puppeteer** — ブラウザ操作

一覧: https://github.com/modelcontextprotocol/servers

---

## まとめ

| 要素 | 説明 |
|---|---|
| MCP | LLMが外部ツールに接続するためのオープンプロトコル |
| MCP Server | ツールを公開する側。`mcp.server.fastmcp` で Python 関数を公開 |
| MCP Client | ツールを使う側。LangChain の langchain-mcp-adapters |
| SSE | Server-Sent Events 通信。Colab 推奨方式 |
| stdio | サブプロセス通信。ローカル環境（Colab 以外）用 |
| HTTP | HTTP通信。リモート・本番用 |
| FastMCP | `@mcp.tool()` でPython関数をMCPツール化。docstringがLLMへの説明になる |
| MultiServerMCPClient | 複数MCPサーバに同時接続。v0.1.0以降は `async with` 不可 |
| create_agent | LangGraph V1.0以降の推奨エージェント作成関数（`from langchain.agents`） |
| uvicorn | FastMCPでポートを変えて起動する場合に使用（`mcp.run()` は `port` 引数不可） |

## 参考リンク

- [LangChain MCP ドキュメント](https://docs.langchain.com/oss/python/langchain/mcp)
- [langchain-mcp-adapters GitHub](https://github.com/langchain-ai/langchain-mcp-adapters)
- [FastMCP ドキュメント](https://gofastmcp.com/)
- [MCP 公式サイト](https://modelcontextprotocol.io/)
- [公開MCPサーバ一覧](https://github.com/modelcontextprotocol/servers)
