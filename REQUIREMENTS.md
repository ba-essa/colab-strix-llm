# Strixlibe × Google Colab 高速GPU LLM 統合 要件定義書

## 1. プロジェクト背景と目的
- **背景**: 自律型ペンテストエージェント「Strixlibe（strix）」において、商用LLM（Gemini, OpenAI等）ではエクスプロイトコード生成や攻撃シナリオ検証時にセーフティガードレール（検閲・拒否）が働き、スキャンが中断または制限される課題がある。
- **目的**:
  1. Google Colab（**L4 / T4 GPU**）を活用し、オープンソース LLM を Colab 上でホストする。
  2. **起動時間の制約（10分以内）** を厳格に遵守するため、4bit AWQ 量子化モデル（8.5GB〜19GB）を採用し、ダウンロード時間を 1〜5 分に短縮する。
  3. L4 GPU（約 2.5〜3.0 CU/h）または T4 GPU（約 1.5〜2.0 CU/h）を用いることで、Colab Pro の 100 CU/月 枠で 35〜65 時間の安定稼働を実現する。
  4. Colab 停止中は、既存の安定した **OpenCode (Zen)** 設定へ自動フォールバックし、本番・検証環境ともに無停止運用を実現する。
  5. アイドリング時の自動停止（CU保護）により、クーポンの浪費を防止する。

---

## 2. システム構成と技術選定

### (1) インフラストラクチャ
- **実行基盤**: Google Colab Pro / AI Pro
- **ハードウェア**:
  - **L4 GPU (24GB VRAM)**: 32B AWQ（約19GB）に最適。消費レート 約 2.5 〜 3.0 CU/h（月35〜40時間稼働）
  - **T4 GPU (16GB VRAM)**: 14B AWQ（約8.5GB）に最適。消費レート 約 1.5 〜 2.0 CU/h（月50〜65時間稼働）
  - **G4 GPU (96GB VRAM)**: より高速な推論が必要な場合に選択可能

### (2) 推論エンジン & モデルラインナップ
- **推論エンジン**: **vLLM** (`--quantization awq`, `--gpu-memory-utilization 0.90`, `--max-model-len 16384`)
- **選択可能モデル（ノートブック上でワンクリック切り替え）**:
  1. `casperhansen/deepseek-r1-distill-qwen-32b-awq`（**既定値**: 約19GB、DL約3〜5分。32B 最高推論、L4推奨）
  2. `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ`（約19GB、DL約3〜5分。32B コーディング特化、L4推奨）
  3. `casperhansen/deepseek-r1-distill-qwen-14b-awq`（約8.5GB、DL約1〜2分！超高速起動・推論思考、T4/L4推奨）
  4. `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ`（約8.5GB、DL約1〜2分！超高速起動・コーディング特化、T4/L4推奨）
  5. `Qwen/Qwen2.5-Coder-7B-Instruct-AWQ`（約5GB、DL約1分！爆速・T4最安）

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
- **Strixlibe スキャン実行時**:
  - Dispatcher が GCE Worker VM を立ち上げる際、Colab LLM のヘルスチェックを実施。
  - **Colab 生存時**: GCE メタデータへ Colab の `LLM_API_BASE` とモデル名を注入。
  - **Colab 停止・未起動時**: 自動的に既存の **OpenCode (Zen)** 設定（`https://opencode.ai/zen/v1` / `OPENCODE_ZEN_API_KEY`）を注入。

### (2) CU 浪費防止（自動シャットダウン）
- **アクセス監視ウォッチドッグ**:
  - Colab 内でバックグラウンドスレッドが直近リクエスト時刻を監視。
  - 直近の推論リクエストから **20分間** リクエストが途絶えた場合、自律的に `google.colab.runtime.unassign()` を実行してインスタンスを切断・解放。
