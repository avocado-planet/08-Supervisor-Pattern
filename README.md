# LangGraph Supervisor パターン

## 概要

Supervisorパターンは、1つの中央Agentが複数の専門Worker Agentを指揮する最も主流なマルチエージェント構成。本資料ではITサポートチケットルーティングシステムを題材に、実装の全体像を解説する。

---

## システム構成

```
ユーザーリクエスト
       │
       ▼
┌─────────────────────────────────────┐
│         Supervisor Agent            │
│  ・リクエスト内容を解析              │
│  ・適切なAgentを選択               │
│  ・最終回答をまとめてユーザーへ返す  │
└────────┬──────────┬──────────┬──────┘
         │          │          │
         ▼          ▼          ▼
   network_agent  account_agent  hardware_agent
   （ネットワーク） （アカウント）  （ハードウェア）
```

---

## 実装ステップ

### Step 1: ツール定義

Worker Agentが使用するツールをPython関数として定義する。`@tool` デコレータを付けるだけでLangChainのツールになる。

```python
from langchain_core.tools import tool

@tool
def check_network_status(location: str) -> str:
    """指定ロケーションのネットワーク状態を確認する。"""
    # 本番では実際のネットワーク監視APIを呼ぶ
    return f"{location}: 正常稼働中"

@tool
def reset_user_password(username: str) -> str:
    """ユーザーのパスワードをリセットし、仮パスワードを発行する。"""
    return f"パスワードリセット完了: [{username}]"
```

**ポイント**: docstringがLLMへのツール説明になるため、何をするツールか明確に書く。

---

### Step 2: Worker Agent の作成

`create_react_agent()` で各専門Agentを作成する。

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent

llm = ChatAnthropic(model="claude-3-5-haiku-20241022")

network_agent = create_react_agent(
    model=llm,
    tools=[check_network_status, create_network_ticket],
    name="network_agent",           # ← Supervisorがこの名前で呼ぶ
    prompt="あなたはネットワーク専門エンジニアです...",
)
```

**`name` パラメータが重要**: Supervisorはこの名前を使ってAgentを指名する。

---

### Step 3: Supervisor の作成

```python
from langgraph_supervisor import create_supervisor
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()  # マルチターン会話用

workflow = create_supervisor(
    agents=[network_agent, account_agent, hardware_agent],
    model=llm,
    prompt=supervisor_prompt,
    output_mode="last_message",   # Supervisorの最終回答のみ返す
)

app = workflow.compile(checkpointer=checkpointer)
```

---

### Step 4: 実行

```python
from langchain_core.messages import HumanMessage

config = {"configurable": {"thread_id": "ticket-001"}}

result = app.invoke(
    {"messages": [HumanMessage(content="大阪支店でネットワークが繋がりません。")]},
    config=config,
)

print(result["messages"][-1].content)
```

---

## create_supervisor の主要パラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|-----------|------|
| `agents` | list | 必須 | 管理するWorker Agentのリスト |
| `model` | LLM | 必須 | SupervisorのLLM |
| `prompt` | str/SystemMessage | None | Supervisorへの指示 |
| `output_mode` | str | `"last_message"` | 出力形式（後述） |
| `add_handoff_messages` | bool | True | handoffメッセージを履歴に含めるか |
| `parallel_tool_calls` | bool | False | 複数AgentへのhandoffをPythonで並列化 |

### output_mode の違い

```
last_message（推奨）:
  ユーザー → Supervisor → Worker A → Worker B → [Supervisorの最終まとめ]
  返却: [Supervisorの最終まとめ] のみ

full_history:
  ユーザー → Supervisor → Worker A → Worker B → [Supervisorの最終まとめ]
  返却: 全メッセージ（Worker内のやりとりも含む）
```

---

## マルチターン会話の仕組み

`InMemorySaver` と `thread_id` の組み合わせで会話継続が可能。

```python
checkpointer = InMemorySaver()
app = workflow.compile(checkpointer=checkpointer)

# 1回目
app.invoke(
    {"messages": [HumanMessage("PCのキーボードが壊れました。")]},
    config={"configurable": {"thread_id": "ticket-004"}},
)

# 2回目: 同じthread_idで続きの会話（前の内容を記憶している）
app.invoke(
    {"messages": [HumanMessage("ユーザーはsuzukiです。")]},
    config={"configurable": {"thread_id": "ticket-004"}},
)
```

---

## handoff の仕組み

SupervisorからWorker Agentへの委譲は **ツール呼び出し** として実装されている。

```
Supervisorの内部ツールリスト:
  - transfer_to_network_agent()   ← 自動生成
  - transfer_to_account_agent()   ← 自動生成
  - transfer_to_hardware_agent()  ← 自動生成
```

LLMが「このリクエストはnetwork_agentが適切」と判断すると、`transfer_to_network_agent()` を呼び出し、制御がWorker Agentに移る。Worker Agentは処理完了後、Supervisorに結果を返す。

カスタムhandoffツールを使う場合:

```python
from langgraph_supervisor import create_handoff_tool

workflow = create_supervisor(
    agents=[network_agent, account_agent],
    tools=[
        create_handoff_tool(
            agent_name="network_agent",
            name="assign_to_network_team",          # ツール名をカスタマイズ
            description="ネットワーク障害・接続問題を担当チームに割り当てる",
        ),
    ],
    model=llm,
)
```

---

## 内部動作の観察方法

`stream()` + `stream_mode="updates"` でノード単位に処理を追える。

```python
for chunk in app.stream(
    {"messages": [HumanMessage(content=query)]},
    config=config,
    stream_mode="updates",
):
    for node_name, node_output in chunk.items():
        print(f"▶ ノード: [{node_name}]")
        # node_name が "supervisor", "network_agent" などになる
```

**出力例（複数Agent振り分け時）:**

```
▶ ノード: [supervisor]        ← ルーティング判断
▶ ノード: [account_agent]     ← アカウント処理
▶ ノード: [supervisor]        ← 結果受け取り・次のAgentを指名
▶ ノード: [network_agent]     ← ネットワーク処理
▶ ノード: [supervisor]        ← 最終まとめ生成
```

---

## Supervisor promptの書き方

Supervisorの振る舞いはpromptで大きく変わる。実務で効果的なパターン:

```python
supervisor_prompt = """
あなたはITサポートデスクのSupervisorです。

担当Agentの選択基準:
- network_agent : ネットワーク・VPN・Wi-Fi・接続に関する問題
- account_agent : パスワード・ログイン・アカウントロック・権限
- hardware_agent: PC・モニター・キーボード・周辺機器

ルール:
1. 複数の問題が含まれる場合は、それぞれ適切なAgentに順番に振り分ける
2. 不明な場合はaccount_agentに振り分ける（デフォルト）
3. すべての対応完了後、ユーザーへの簡潔なサマリーを日本語で返す
"""
```

**promtのポイント:**
- 各Agentの担当を明確に記述する
- 複数問題の処理方針を明示する
- デフォルトのフォールバックAgentを指定する
- 最終回答のフォーマットを指定する

---

## 実装時の注意点

### Worker AgentのnameはSnake_case

```python
# ✅ 正しい
create_react_agent(name="network_agent")

# ❌ スペースや特殊文字は避ける
create_react_agent(name="network agent")
```

### Supervisorのpromptは具体的に

LLMはpromptの曖昧さに敏感。担当Agentの選択基準が曖昧だと誤ルーティングが発生しやすい。

### output_modeはlast_messageを推奨

`full_history` はデバッグ用。本番では `last_message` で不要な情報を除外し、コンテキストウィンドウを節約する。

### add_handoff_messages=False で履歴を整理

Workerへの委譲メッセージ（"Transferring to network_agent..."）は最終ユーザーには不要なため、Falseにすることでノイズを減らせる。

---

## よく使うパターン集

### パターン1: シンプルなルーティング（本資料のメイン）

```python
workflow = create_supervisor(
    agents=[agent_a, agent_b, agent_c],
    model=llm,
    prompt="...",
)
app = workflow.compile()
```

### パターン2: メモリ付き（マルチターン）

```python
app = workflow.compile(checkpointer=InMemorySaver())
# invoke時にthread_idを指定
```

### パターン3: 長期ストア付き

```python
from langgraph.store.memory import InMemoryStore
store = InMemoryStore()
app = workflow.compile(checkpointer=InMemorySaver(), store=store)
```

### パターン4: カスタムhandoffツール

```python
from langgraph_supervisor import create_handoff_tool
workflow = create_supervisor(
    agents=[...],
    tools=[create_handoff_tool(agent_name="agent_a", description="...")],
    model=llm,
)
```

---

## まとめ

| 概念 | 役割 |
|------|------|
| Supervisor | リクエストを解析し適切なWorkerに委譲・最終回答を生成 |
| Worker Agent | 特定ドメインの処理を担当・結果をSupervisorに返す |
| handoff tool | SupervisorからWorkerへの制御委譲の仕組み |
| checkpointer | マルチターン会話のための状態永続化 |
| thread_id | 会話スレッドの識別子 |
| output_mode | 返却するメッセージの範囲を制御 |

---

## 参考

- [langgraph-supervisor GitHub](https://github.com/langchain-ai/langgraph-supervisor-py)
- [LangGraph Supervisor Tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/)
- [Benchmarking Multi-Agent Architectures](https://blog.langchain.com/benchmarking-multi-agent-architectures/)
