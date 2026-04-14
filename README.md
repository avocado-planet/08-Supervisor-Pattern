# LangGraph Supervisor パターン


## 2つの実装方法

同じITサポートルーティングシステムを2通りで実装できる。

```
ユーザーリクエスト
       │
       ▼
  Supervisor Agent
   ├─→ network_agent  （ネットワーク障害・接続）
   ├─→ account_agent  （アカウント・認証）
   └─→ hardware_agent （PC・周辺機器）
```

---

## 方法A: langgraph-supervisor ライブラリ

### 特徴

- `create_supervisor()` にWorker Agentのリストを渡すだけ
- handoff toolが自動生成される
- 素早くプロトタイプを作るのに向いている

### 制御フロー

```
Supervisor
  └─[transfer_to_network_agent]→ network_agent
                                      └─→ Supervisor（結果を返す）
```

### 実装

```python
# インストール: pip install langgraph-supervisor langchain langchain-anthropic
from langchain.agents import create_agent          # ✅ 新API
from langchain_anthropic import ChatAnthropic
from langgraph_supervisor import create_supervisor
from langgraph.checkpoint.memory import InMemorySaver

llm = ChatAnthropic(model="claude-3-5-haiku-20241022")

# Worker Agent（新APIで作成）
network_agent = create_agent(
    model=llm,
    tools=[check_network_status, create_network_ticket],
    system_prompt="あなたはネットワーク専門エンジニアです...",
)
account_agent = create_agent(model=llm, tools=[...], system_prompt="...")
hardware_agent = create_agent(model=llm, tools=[...], system_prompt="...")

# Supervisor（ライブラリ版）
workflow = create_supervisor(
    agents=[network_agent, account_agent, hardware_agent],
    model=llm,
    prompt="あなたはITサポートSupervisorです...",
    output_mode="last_message",
)
app = workflow.compile(checkpointer=InMemorySaver())

# 実行
result = app.invoke(
    {"messages": [HumanMessage(content="大阪支店でネットワークが繋がりません。")]},
    config={"configurable": {"thread_id": "ticket-001"}},
)
print(result["messages"][-1].content)
```

---

## 方法B: ツール経由の手動実装（LangChain公式推奨）

### 特徴

- Worker AgentをPythonの `@tool` でラップし、Supervisorの「ツールリスト」に追加する
- `langgraph-supervisor` ライブラリ不要（`langchain` のみで完結）
- Supervisorに渡す情報（コンテキスト）をコードで明示的に制御できる

### 制御フロー

```
Supervisor
  └─[call_network_agent(request="大阪支店の問題...")]→ 結果が直接返る
```

方法Aのように「制御を渡す→受け取る」という往復がなく、
SupervisorがWorkerを関数のように呼び出す。

### なぜこれが推奨か

方法Aの問題（telephone game）:

```
ユーザー: 「大阪支店のパケットロス問題を調査してください」
  ↓
Supervisor: 「network_agentへ転送します」
  ↓
network_agent: 「パケットロス15%を確認しました。チケットNET-1234を起票しました」
  ↓
Supervisor（言い換え）: 「ネットワーク障害が確認され、対応中です」  ← 情報が欠落・変質
```

方法Bでは Worker の回答がそのまま Supervisor のツール結果として返るため、
情報の欠落が発生しにくい。

### 実装

```python
# インストール: pip install langchain langchain-anthropic
from langchain.agents import create_agent
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langchain_anthropic import ChatAnthropic
from langgraph.checkpoint.memory import InMemorySaver

llm = ChatAnthropic(model="claude-3-5-haiku-20241022")

# Step 1: Worker Agentを作成（方法Aと同じ）
network_agent = create_agent(model=llm, tools=[...], system_prompt="...")
account_agent = create_agent(model=llm, tools=[...], system_prompt="...")
hardware_agent = create_agent(model=llm, tools=[...], system_prompt="...")

# Step 2: Worker AgentをPythonの @tool でラップ
@tool
def call_network_agent(request: str) -> str:
    """
    ネットワーク専門Agentに問い合わせる。
    ネットワーク障害、接続問題、VPN、Wi-Fiに関するリクエストに使用。
    request: ネットワーク問題の詳細説明
    """
    result = network_agent.invoke(
        {"messages": [HumanMessage(content=request)]}
    )
    return result["messages"][-1].content

@tool
def call_account_agent(request: str) -> str:
    """
    アカウント管理専門Agentに問い合わせる。
    パスワードリセット、アカウントロック解除に使用。
    request: アカウント問題の詳細（ユーザー名を含めること）
    """
    result = account_agent.invoke(
        {"messages": [HumanMessage(content=request)]}
    )
    return result["messages"][-1].content

@tool
def call_hardware_agent(request: str) -> str:
    """
    ハードウェア専門Agentに問い合わせる。
    PC・モニター・キーボード・マウスの問題に使用。
    request: ハードウェア問題の詳細（ユーザー名と機器種類を含めること）
    """
    result = hardware_agent.invoke(
        {"messages": [HumanMessage(content=request)]}
    )
    return result["messages"][-1].content

# Step 3: Supervisorは「ツールとしてのWorker Agent」を持つ普通のcreate_agent
supervisor = create_agent(
    model=llm,
    tools=[call_network_agent, call_account_agent, call_hardware_agent],
    system_prompt="""
あなたはITサポートデスクのSupervisorです。

ツールの使い分け:
- call_network_agent : ネットワーク・VPN・Wi-Fiの問題
- call_account_agent : パスワード・アカウントロック・権限の問題
- call_hardware_agent: PC・モニター・キーボードの問題
""",
)

app = supervisor.compile(checkpointer=InMemorySaver())

# 実行（方法Aと同じ呼び出し方）
result = app.invoke(
    {"messages": [HumanMessage(content="大阪支店でネットワークが繋がりません。")]},
    config={"configurable": {"thread_id": "ticket-001"}},
)
print(result["messages"][-1].content)
```

---

## 方法A vs 方法B 比較

| 観点 | 方法A (langgraph-supervisor) | 方法B（推奨） |
|------|------------------------------|--------------|
| 追加パッケージ | `langgraph-supervisor` 必要 | `langchain` のみ |
| コード量 | 少ない | やや多い |
| handoff仕組み | 自動生成（ブラックボックス） | `@tool` で明示的 |
| コンテキスト制御 | ライブラリ任せ | コードで明示 |
| 情報欠落リスク | やや高い | 低い |
| カスタマイズ性 | 制限あり | 無制限 |
| 向いている場面 | プロトタイプ・検証 | 本番・複雑なフロー |

---

## create_agent の主要パラメータ

```python
from langchain.agents import create_agent

agent = create_agent(
    model=llm,                    # LLMインスタンス
    tools=[tool1, tool2],         # ツールリスト
    system_prompt="...",          # システムプロンプト（文字列）
    # その他オプション
    # middleware=[...],           # ミドルウェア（新機能）
    # checkpointer=...,           # 直接渡す場合
)

# compile() でチェックポインターを付与
app = agent.compile(checkpointer=InMemorySaver())
```

`create_agent` は `create_react_agent` との主な違い:

| | `create_react_agent`（非推奨） | `create_agent`（推奨） |
|--|-------------------------------|----------------------|
| インポート | `langgraph.prebuilt` | `langchain.agents` |
| システムプロンプト引数 | `prompt=` | `system_prompt=` |
| ミドルウェア | なし | `middleware=` で対応 |
| 状態 | 非推奨（後方互換維持） | 現在の推奨 |

---

## マルチターン会話

```python
# InMemorySaver でセッション間の状態を保持
app = agent.compile(checkpointer=InMemorySaver())

# 同じ thread_id を使うと前の会話を記憶
config = {"configurable": {"thread_id": "ticket-004"}}

# 1回目
app.invoke({"messages": [HumanMessage("キーボードが壊れました。")]}, config=config)

# 2回目: 前の内容を覚えている
app.invoke({"messages": [HumanMessage("ユーザーはsuzukiです。")]}, config=config)
```

---

## 内部動作の観察

```python
for chunk in app.stream(
    {"messages": [HumanMessage(content=query)]},
    config=config,
    stream_mode="updates",
):
    for node_name, node_output in chunk.items():
        print(f"▶ ノード: [{node_name}]")
        # 方法Bでは call_network_agent などのツール名がそのまま見える
```

---

## ツールのdocstring が重要

方法Bでは `@tool` のdocstringがSupervisorのツール説明になる。
SupervisorのLLMがこの説明を読んで「どのツールを呼ぶか」を判断するため、
明確な説明を書くことがルーティング精度に直結する。

```python
# ❌ 曖昧な説明
@tool
def call_network_agent(request: str) -> str:
    """ネットワークの問題を処理する。"""
    ...

# ✅ 明確な説明（いつ使うか、何を渡すかを明記）
@tool
def call_network_agent(request: str) -> str:
    """
    ネットワーク専門Agentに問い合わせる。
    ネットワーク障害、接続問題、VPN、Wi-Fiに関するリクエストに使用すること。
    request: ネットワーク問題の詳細説明（ロケーション情報を含めること）
    """
    ...
```

---

## 参考

- [LangChain v1.0 リリースノート](https://blog.langchain.com/langchain-langgraph-1dot0/)
- [Build a personal assistant with subagents（公式チュートリアル）](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents-personal-assistant)
- [langgraph-supervisor GitHub](https://github.com/langchain-ai/langgraph-supervisor-py)
