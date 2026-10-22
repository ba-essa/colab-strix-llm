# 🦉 Strixlibe × Google Colab G4 GPU (Blackwell 96GB) LLM Server

Google Colab 最新の **G4 GPU (NVIDIA RTX PRO 6000 Blackwell 96GB VRAM)** 上で、最高峰オープンソース LLM（`DeepSeek-R1-Distill-Qwen-32B` BF16 フル精度、`Qwen2.5-72B`、`Llama-3.3-70B` 等）を自律ホストし、自律ペンテストエージェント「Strixlibe」と完全自動同期するプロジェクトです。

---

## ⚡ G4 (Blackwell 96GB) の特長

- **96GB 大容量 VRAM**: 32B モデルを量子化なしの BF16 フル精度（劣化ゼロ）で動かせるほか、72B クラスのフラッグシップモデルも余裕で稼働。
- **省CUレート**: 約 6.4 〜 8.7 CU/h（A100 より約30%省エネ、月100CU枠で約12〜15時間稼働可能）。
- **セーフティガードレール（検閲）の回避**: 商用APIで拒否されがちな攻撃コードやエクスプロイト検証を制約なく実行。
- **ゼロコンフィグ自動同期**: Colab 起動時は自動で Strixlibe のルーティング先になり、停止時は既存の **OpenCode (Zen)** 設定へ自動フォールバック。
- **CU保護ウォッチドッグ**: スキャン完了後、20分間無通信で `runtime.unassign()` によりインスタンスを自動切断。

---

## 🚀 Google Colab で直接開く

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ba-essa/colab-strix-llm/blob/main/colab_strix_llm_server.ipynb)

---

## 📁 ディレクトリ構成

- `/home/ada/agy/colab-strix-llm/`
  - `README.md`: 本ドキュメント（概要とクイックスタート）
  - `REQUIREMENTS.md`: G4 対応の詳細要件定義書
  - `ARCH_DESIGN.md`: Dispatcher 連携・Egress Guard・フォールバック設計書
  - `colab_strix_llm_server.ipynb`: G4 最適化ワンクリック実行ノートブック

---

## 📋 選択可能モデル一覧（ノートブックのフォームで切替可能）

1. `deepseek-ai/DeepSeek-R1-Distill-Qwen-32B`（**既定値**: BF16 フル精度、推論思考チェーンによる最高精度）
2. `Qwen/Qwen2.5-72B-Instruct-AWQ`（72B 最大規模パラメータ、GPT-4o級の推論力）
3. `casperhansen/llama-3.3-70b-instruct-awq`（70B Llama 3.3）
4. `casperhansen/deepseek-r1-distill-qwen-32b-awq`（32B AWQ、ロード1分 & 64k超長文丸呑み）
5. `Qwen/Qwen2.5-Coder-32B-Instruct`（32B コーディング特化 BF16 フル精度）
