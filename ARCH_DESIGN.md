# Strixlibe × Colab LLM アーキテクチャ設計書

## 1. 全体アーキテクチャ概要

```
[Google Colab (A100)]
  ├── vLLM (Port 8000: DeepSeek-R1-Distill-Qwen-32B-AWQ)
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
  │     ├── [Active]  -> GCE メタデータ: Colab Tunnel URL
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
    "provider": "colab-vllm",
    "base_url": "https://xxxx-xx-xx-xx.trycloudflare.com/v1",
    "model_name": "casperhansen/deepseek-r1-distill-qwen-32b-awq",
    "env_target": "all",
    "ttl_minutes": 60
  }
  ```
- **レスポンス**:
  ```json
  {
    "status": "registered",
    "active_llm": "colab-vllm",
    "fallback_llm": "opencode/zen",
    "expires_at": "2026-10-22T14:30:00Z"
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
- これにより、Worker VM 側のコード改修は不要で、Dispatcher の設定注入ロジックのみで完全対応可能。

---

## 4. CU保護ウォッチドッグ仕様（Colab側）

- vLLM プロキシ側で最後のリクエストタイムスタンプ（`last_request_time`）をメモリ保持。
- 60秒ごとにバックグラウンドスレッドがチェック。
- `current_time - last_request_time > 1200`（20分）かつ `idle_count >= 1` の場合：
  1. Dispatcher に `DELETE /api/v1/llm-session` を送信（Colab 終了を通知）。
  2. Slack に「🛑 アイドリング検出のため Colab を自動切断しました」と通知。
  3. `from google.colab import runtime; runtime.unassign()` を呼び出し、GPU コンピュートユニットの課金を即時停止。
