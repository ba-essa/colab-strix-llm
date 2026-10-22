# 🦉 Strixlibe × Google Colab GPU LLM Server

Google Colab（AI Pro / Colab Pro の A100/L4 GPU）上で高性能オープンソース LLM（`DeepSeek-R1-Distill-Qwen-32B` 等）を自律ホストし、自律ペンテストエージェント「Strixlibe」と完全自動同期するプロジェクトです。

---

## 💡 プロジェクトの特徴

- **セーフティガードレール（検閲）の回避**:
  - 商用API（Vertex AI / Gemini / OpenAI）で拒否されがちな高度な脆弱性・エクスプロイト検証を、オープンウェイトモデルで制約なく実行。
- **本番・検証環境のゼロコンフィグ自動同期**:
  - Colab 起動時: Cloudflare Quick Tunnel 経由で Dispatcher へ自動登録。Strixlibe のスキャンが自動で Colab LLM にルーティング。
  - Colab 停止時: 既存の **OpenCode (Zen)** 設定（`https://opencode.ai/zen/v1`）へ自動フォールバック。手動での設定変更は不要。
- **コンピュートユニット（CU）保護**:
  - 診断完了後、20分間リクエストが途絶えると `runtime.unassign()` により Colab インスタンスを自動切断。クーポンの浪費・枯渇を防止。
- **アカウント不要トンネル**:
  - Cloudflare Quick Tunnel (`cloudflared`) により、トークンやアカウント登録なしで即座にセキュアな OpenAI 互換エンドポイントを発行。

---

## 🚀 Google Colab で直接開く

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ba-essa/colab-strix-llm/blob/main/colab_strix_llm_server.ipynb)
