---
title: "Arona"
description: "AI モデルのセルフデプロイメントとリモート管理のためのプラットフォーム — gateway、backend、billing、agent、memory。"
---

# Arona

**AI モデルのセルフデプロイメントとリモート管理のためのプラットフォーム。**

Arona は Rust（axum）で書かれた **純粋な backend** プラットフォームです。自前のハードウェアで稼働させるモデルに対する OpenAI 互換のモデル gateway *であると同時に*、管理プレーンでもあります。`/v1/*` の OpenAI 互換 REST API、JSON-RPC 2.0 管理プレーン（`/api/rpc`）、エージェント制御プレーン（`/ws/agent`）、そして `/docs` の Swagger UI を提供します。

**バンドルされた Web ダッシュボードや CLI チャットはありません** — チャット + 管理 UI は [shittim-chest](https://github.com/celestia-island/shittim-chest) にあり、RPC 経由で Arona と通信します。Arona はサーバー側に集中しています。ルーティング、billing、auth、モデルデプロイ、agent、memory です。

## 機能マトリクス

| 領域 | Arona が提供するもの |
| --- | --- |
| **会話コア** | OpenAI 互換の `chat.completions`（ストリーミング + 非ストリーミング）、`embeddings`、`models` 一覧。最後に `[DONE]` チャンクで終わるストリーミングと、最終チャンクでの実際の usage を提供します。 |
| **Backend** | 管理者が登録する upstream。`external`（任意の OpenAI 互換 HTTP API）、`ollama`、CEP `engine`（WebSocket）、`minimax-cloud` ビデオ、そして産業用・エッジサービスへの `evernight://` ブリッジ URL。 |
| **認証** | JWT access/refresh ペア（15 分 / 7 日）、SHA-256 ハッシュで保存される API key `arona-{uuid}`、3 段階の admin 権限、パスワードポリシー、二系統のレート制限。 |
| **Billing & usage** | シード済みの tier（free / pro / enterprise）、全チャネルでのリクエスト単位の usage レコード、plana の価格テーブル、プロジェクト単位のクォータスコープ、429 + `Retry-After`。 |
| **モデル管理** | モデルダウンロード（`hf:` / `ms:` / `gh:` ソース）、`_agent` GPU ノードへのデプロイ、デプロイ済みモデルのルーティング可能な backend としての自動登録。 |
| **Realtime & マルチモーダル** | 全二重の `realtime.*` セッション、`engine.invoke` 認識・制御チャネル、非同期ビデオ生成ジョブ（MiniMax cloud）。 |
| **Agent クラスター** | GPU ノードが `/ws/agent` 経由で接続、最少負荷配置、セッションアフィニティ、再起動をまたぐノード永続化。 |
| **Memory gateway** | entelecheia Philia による長期メモリ。リコール注入、エピソードの書き戻し、明示的な劣化動作。 |
| **運用** | ヘルスプローブ、`RUST_LOG` トレーシング、upstream エラーマッピング（502 vs 500）、グレースフルシャットダウン、起動時の自動マイグレーション。 |

## 位置づけ

Arona は **gateway + プラットフォーム** です。モデルトラフィックを backend にルーティングし、モデルを GPU agent にデプロイし、すべてを計測します。

- **pi** との比較 — pi はモデルと会話する CLI アシスタントで、arona に CLI チャットはありません。Arona は pi（および他のツール）が通信する*相手*となるプラットフォームです。
- **one-api / new-api** との比較 — これらはモデルproviderの前に立つ API key gateway です。arona はそれに加えて **モデルデプロイ**（重みのダウンロード、agent 上での実行）、完全な管理 RPC プレーン、billing tier、memory を備えています。
- **LiteLLM** との比較 — gateway としては同等です。arona はさらに、gateway の背後にあるモデルのデプロイライフサイクルを自前で管理します。

## ここから始める

- [クイックスタート](quickstart.md) — 組み込みの mock upstream によるエンドツーエンド。
- [設定](configuration.md) — すべての環境変数。
- [デプロイ](deployment.md) — ベアメタル、systemd、Docker、監視。
- [Backends](backends.md) — backend タイプ、URL セマンティクス、プロービング。
- [OpenAI 互換 REST API](../api/openai-rest.md) — `/v1/*`。
- [JSON-RPC API](../api/jsonrpc.md) — 完全な管理プレーン。

## リポジトリレイアウト

```
packages/
├── core/    # Core logic: gateway, routing, billing, auth, memory, RPC
├── agent/   # Remote agent (bin `_agent`): deployed on GPU machines
└── cli/     # CLI (bin `_cli`): install, deploy, serve, migrate, download
```

Web ダッシュボードはこのリポジトリから削除され、[shittim-chest](https://github.com/celestia-island/shittim-chest)（chest #291）に移りました。

<!-- src: README.md (repo root) / packages/core/src/gateway/server.rs:128-163 (route table) -->
