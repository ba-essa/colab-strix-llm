# Strixlibe × Google Colab GPU LLM 統合 要件定義書

## 1. プロジェクト背景と目的
- **背景**: 自律型ペンテストエージェント「Strixlibe（strix）」において、商用LLM（Gemini, OpenAI等）ではエクスプロイトコード生成や攻撃シナリオ検証時にセーフティガードレール（検閲・拒否）が働き、スキャンが中断または制限される課題がある。
- **目的**:
  1. Google One AI Pro / Colab 契約の GPU クレジット（コンピュートユニット）を有効活用し、オープンソース LLM を Colab 上でホストする。
  2. ガードレールのない高性能推論モデルにより、高度な脆弱性・エクスプロイト検証を制限なく実行可能にする。
  3. Colab 停止中は、既存の安定した **OpenCode (Zen)** 設定へ自動フォールバックし、本番・検証環境ともに無停止運用を実現する。
  4. コンピュートユニットの浪費を防ぐため、アイドリング時の自動停止（CU保護）を徹底する。

---

## 2. システム構成と技術選定

### (1) インフラストラクチャ
- **実行基盤**: Google Colab Pro / AI Pro
- **ハードウェア**: NVIDIA A100 GPU (40GB / 80GB VRAM) または L4 GPU (24GB)
  - 集中検証モード: 1回1〜2時間、月数回の大型診断（100 CU/月 の範囲で十分運用可能）

### (2) 推論エンジン & モデル
- **推論エンジン**: **vLLM** (PagedAttention & OpenAI互換 API `/v1/chat/completions`)
- **既定モデル**: `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B` (AWQ量子化版: `casperhansen/deepseek-r1-distill-qwen-32b-awq`)
  - 選定理由: R1 の推論・思考チェーン（Thinking Process）により、多段階の攻撃シナリオや脆弱性の根本原因分析に極めて高い精度を発揮。
  - 代替切替モデル: `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ`（コーディング特化）もノートブック上で選択可能。

### (3) トンネリング & ネットワーク
- **トンネル技術**: **Cloudflare Quick Tunnel (`cloudflared`)**
  - アカウント登録・APIトークン不要。
  - Colab 起動時にコマンド1行で一時的な一意の公開URL（`https://<random>.trycloudflare.com`）を発行。
  - GCE Ephemeral Worker VM の Egress Guard (`egress_guard.py`) が `LLM_API_BASE` のFQDNを自動的に iptables ホワイトリストに追加。

---

## 3. 本番・検証環境の自動同期 & フォールバック要件

### (1) 稼働状態の動的検知
- **Colab 起動時**:
  - vLLM と Cloudflare Tunnel が確立されたら、GCP Cloud Run Dispatcher の内部エンドポイント（`/config/llm-session`）へトンネルURLとモデル名を自動登録（POST）。
  - Dispatcher 側でセッションIDと有効期限（TTL）を管理。
- **Strixlibe スキャン実行時**:
  - Dispatcher が GCE Worker VM を立ち上げる際、Colab LLM のヘルスチェックを実施。
  - **Colab 生存時**: GCE メタデータへ Colab の `LLM_API_BASE` とモデル名を注入。
  - **Colab 停止・未起動時**: 自動的に既存の **OpenCode (Zen)** 設定（`https://opencode.ai/zen/v1` / `OPENCODE_ZEN_API_KEY`）を注入。
- **本番（`scan.ba-essa.com`）と検証（`strix-wrapper-staging`）の両立**:
  - Colab からの登録時に環境フラグ（`env: "all"` / `"staging"` / `"production"`）を指定可能とし、検証環境のみで先行テストすることも可能。

### (2) CU 浪費防止（自動シャットダウン）
- **アクセス監視ウォッチドッグ**:
  - Colab 内でプロキシ/ログ監視プロセスをバックグラウンド実行。
  - 直近の推論リクエストから **20分間** リクエストが途絶えた場合、自律的に `google.colab.runtime.unassign()` を実行してインスタンスを切断・解放。
  - これにより、スキャン完了後の放置によるコンピュートユニットの全損を確実に防止。

---

## 4. 運用フロー

```
【ユーザー操作】
1. ブラウザで Colab ノートブックを開く
2. 「すべてのセルを実行（Run All）」をクリック（約3分で起動完了）
   │
   ├─► vLLM 起動 (A100)
   ├─► Cloudflare Tunnel 確立
   ├─► Dispatcher へアクティブURLを自動登録
   ├─► Slack へ「🚀 Colab LLM起動完了」通知
   └─► 20分無通信タイマー待機
   │
【Strixlibe 実行】
3. 本番または検証環境でスキャンを実行（自動で Colab LLM が使用される）
   │
【完了・自動後処理】
4. スキャン完了後、20分経過すると Colab が自動停止（CU浪費ゼロ）
5. 次回スキャン時は、Colab が無ければ自動で OpenCode (Zen) で動作
```
