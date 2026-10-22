# Strixlibe × Colab G4 LLM アーキテクチャ設計書

## 1. 全体アーキテクチャ概要

```
[Google Colab (G4 GPU: Blackwell 96GB)]
  ├── vLLM (Port 8000: DeepSeek-R1-32B BF16 / Qwen2.5-72B-AWQ)
  ├── Watchdog (20分無通信で runtime.unassign())
  └── Cloudflare Quick Tunnel (cloudflared)
         │
         ▼ HTTPS
   https://<id>.trycloudflare.com/v1
         ▲
         │ (自動同期POST)
         ▼
[GCP Cloud Run Dispatcher] ── (ハートビート・セッション保持)
  │
  ├── ジョブ受信時: Colab がアクティブか判定
  │     ├── [Active]  -> GCE メタデータ: Colab Tunnel URL (G4 Blackwell)
  │     └── [Offline] -> GCE メタデータ: OpenCode (Zen) 設定
  │
  ▼ Compute Engine API
[GCE Ephemeral Worker VM]
  ├── Egress Guard (Colab FQDN または OpenCode API をホワイトリスト化)
  └── Strix Engine (LiteLLM 経由で推論実行)
```

---

## 2. インターフェース仕様

### (1) Dispatcher 登録エンドポイント
- **パス**: `POST /api/v1/llm-session`
- **認証**: `X-API-Key` (既存の `GCP_DISPATCHER_API_KEY`)
- **リクエストボディ**:
  ```json
  {
    "provider": "colab-g4-vllm",
    "base_url": "https://xxxx-xx-xx-xx.trycloudflare.com/v1",
    "model_name": "deepseek-ai/DeepSeek-R1-Distill-Qwen-32B",
    "env_target": "all",
    "ttl_minutes": 120
  }
  ```
- **レスポンス**:
  ```json
  {
    "status": "registered",
    "active_llm": "colab-g4-vllm",
    "fallback_llm": "opencode/zen",
    "expires_at": "2026-10-22T15:30:00Z"
  }
  ```

### (2) セッション解除（明示的シャットダウン時）
- **パス**: `DELETE /api/v1/llm-session`
- **動作**: 登録された Colab URL を破棄し、即座に OpenCode (Zen) をアクティブにする。

---

## 3. GCE Worker 側の自動対応（既存設計との整合性）

- `gcp-worker/egress_guard.py`:
  - `LLM_API_BASE` が渡された場合、自動的にホスト名（例: `xxxx.trycloudflare.com`）を抽出し、Port 443 の外部送信を iptables ホワイトリストに追加する処理が実装済み。
- `gcp-worker/worker_runner.py`:
  - メタデータの `LLM_API_BASE` および `STRIX_LLM` を環境変数として Strix CLI にそのまま渡す処理が実装済み。

---

## 4. CU保護ウォッチドッグ仕様（Colab側）

- バックグラウンドデーモンスレッドが直近リクエスト時刻を監視。
- `current_time - last_seen_time > 1200`（20分）の場合：
  1. Dispatcher に `DELETE /api/v1/llm-session` を送信（Colab 終了を通知）。
  2. Slack に「🛑 アイドリング検出のため Colab G4 を自動切断しました」と通知。
  3. `from google.colab import runtime; runtime.unassign()` を呼び出し、GPU コンピュートユニットの課金を即時停止。
