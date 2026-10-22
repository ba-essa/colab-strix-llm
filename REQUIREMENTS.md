# Strixlibe × Google Colab G4 GPU LLM 統合 要件定義書

## 1. プロジェクト背景と目的
- **背景**: 自律型ペンテストエージェント「Strixlibe（strix）」において、商用LLM（Gemini, OpenAI等）ではエクスプロイトコード生成や攻撃シナリオ検証時にセーフティガードレール（検閲・拒否）が働き、スキャンが中断または制限される課題がある。
- **目的**:
  1. Google Colab 最新の **G4 GPU (NVIDIA RTX PRO 6000 Blackwell 96GB VRAM)** を活用し、オープンソース LLM を Colab 上でホストする。
  2. 96GB の広大な VRAM により、32B モデルの BF16 フル精度（量子化による劣化ゼロ）や 72B クラスのフラッグシップモデルを制限なく実行可能にする。
  3. G4 は約 6.4〜8.7 CU/h と A100 より約30%省エネであり、月 100 CU 枠で約 12〜15 時間の大型診断を実現する。
  4. Colab 停止中は、既存の安定した **OpenCode (Zen)** 設定へ自動フォールバックし、本番・検証環境ともに無停止運用を実現する。
  5. アイドリング時の自動停止（CU保護）により、クーポンの浪費を防止する。

---

## 2. システム構成と技術選定

### (1) インフラストラクチャ
- **実行基盤**: Google Colab Pro / AI Pro
- **ハードウェア**: **G4 GPU** (NVIDIA RTX PRO 6000 Blackwell Server Edition, **96GB GDDR7 VRAM**)
  - 消費レート: 約 6.4 〜 8.7 CU / 時間（月100 CUで約12〜15時間稼働可能）
  - 特徴: GDDR7 広帯域、FP8 / FP4 ネイティブ対応、96GB 大容量による長大コンテキスト（32k〜64k+）の余裕保持

### (2) 推論エンジン & モデルラインナップ
- **推論エンジン**: **vLLM** (`--gpu-memory-utilization 0.92`, `--dtype auto`, `--max-model-len 32768`)
- **選択可能モデル（ノートブック上でワンクリック切り替え）**:
  1. `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`（**既定値**: BF16 フル精度、VRAM 約64GB。推論思考チェーンによる最高峰の脆弱性分析、量子化劣化なし）
  2. `Qwen/Qwen2.5-72B-Instruct-AWQ`（72B パラメータ最大規模、VRAM 約42GB。GPT-4o/Claude 3.5級の洞察力）
  3. `casperhansen/llama-3.3-70b-instruct-awq`（70B Llama 3.3、VRAM 約40GB）
  4. `casperhansen/deepseek-r1-distill-qwen-32b-awq`（32B AWQ、VRAM 約20GB。超高速ロード & 64k超の長文丸呑み）
  5. `Qwen/Qwen2.5-Coder-32B-Instruct`（32B コーディング特化 BF16 フル精度、VRAM 約64GB）

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
  - Colab 内でバックグラウンドスレッドが直近リクエスト時刻を監視。
  - 直近の推論リクエストから **20分間** リクエストが途絶えた場合、自律的に `google.colab.runtime.unassign()` を実行してインスタンスを切断・解放。
  - スキャン完了後の放置によるコンピュートユニットの全損を確実に防止。

---

## 4. 運用フロー

```
【ユーザー操作】
1. ブラウザで Colab ノートブックを開く (G4 GPU 選択)
2. 「すべてのセルを実行（Run All）」をクリック（約3分で起動完了）
   │
   ├─► vLLM 起動 (G4 96GB: DeepSeek-R1 32B フル精度 / 72B)
   ├─► Cloudflare Tunnel 確立
   ├─► Dispatcher へアクティブURLを自動登録
   ├─► Slack へ「🚀 Colab G4 LLM起動完了」通知
   └─► 20分無通信タイマーバックグラウンド待機
   │
【Strixlibe 実行】
3. 本番または検証環境でスキャンを実行（自動で Colab LLM が使用される）
   │
【完了・自動後処理】
4. スキャン完了後、20分経過すると Colab が自動停止（CU浪費ゼロ）
5. 次回スキャン時は、Colab が無ければ自動で OpenCode (Zen) で動作
```
