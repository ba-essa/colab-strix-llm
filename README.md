# 🦉 Strixlibe × Google Colab GPU LLM Server (Ollama 版)

Google Colab（**T4 / L4 / G4 GPU**）上で、Python/CUDAのメモリ衝突や依存関係エラーを起こさない **Ollama** を使って、オープンソース LLM（`DeepSeek-R1 14B/32B`、`Qwen2.5-Coder 14B/7B` 等）を自律ホストし、自律ペンテストエージェント「Strixlibe」と完全自動同期するプロジェクトです。

---

## ⚡ Ollama 方式の圧倒的なメリット

- **Python/PyTorch 依存ゼロ**: ネイティブ C++ バイナリで動くため、vLLM のような CUDA メモリ初期化エラー（OOM）や Python 3.13 の競合が一切起きません。
- **ダウンロード進捗が 100% 見える**: `ollama pull` により、パーセント表示（%）とダウンロード速度がリアルタイムで画面に表示されます（1〜2分で完了）。
- **メモリ安全**: GPU VRAM に合わせて動的にメモリを管理するため、起動時の即死クラッシュが起きません。
- **完全な OpenAI 互換 API**: `https://<tunnel-url>/v1` で Strixlibe（LiteLLM）や Dispatcher からそのまま利用可能。
- **CU保護ウォッチドッグ**: スキャン完了後、20分間無通信で `runtime.unassign()` によりインスタンスを自動切断。

---

## 🚀 Google Colab で直接開く

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ba-essa/colab-strix-llm/blob/main/colab_strix_llm_server.ipynb)

---

## 📋 選択可能モデル一覧

1. `deepseek-r1:14b`（**既定値**: 約9GB、DL約1〜2分。R1推論思考チェーン、T4/L4推奨）
2. `qwen2.5-coder:14b`（約9GB、DL約1〜2分。コーディング・ペンテスト特化、T4/L4推奨）
3. `qwen2.5-coder:7b`（約4.7GB、DL約30秒！最速・T4最安）
4. `deepseek-r1:32b`（約19GB、DL約3〜5分。L4/G4推奨）
5. `qwen2.5-coder:32b`（約19GB、DL約3〜5分。L4/G4推奨）
