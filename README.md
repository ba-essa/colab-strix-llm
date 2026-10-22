# 🦉 Strixlibe × Google Colab 高速GPU LLM Server

Google Colab（**L4 / T4 / G4 GPU**）上で、実測 2〜5 分（10分以内）で高速起動するオープンソース LLM（`DeepSeek-R1-32B/14B AWQ`、`Qwen2.5-Coder-32B/14B AWQ` 等）を自律ホストし、自律ペンテストエージェント「Strixlibe」と完全自動同期するプロジェクトです。

---

## ⚡ 高速ロード & 省CU設計（実測 2〜5分で起動）

- **4bit AWQ 量子化**: 精度劣化 1% 未満のままファイルサイズを 8.5GB〜19GB に圧縮。ダウンロードが数分で完了し、10分以内の起動を確実に達成します。
- **適したGPUの選択**:
  - **L4 GPU (24GB)**: 32B AWQ（約19GB）に最適。CU消費は約 2.5〜3.0 CU/h（月35〜40時間稼働可能）。
  - **T4 GPU (16GB)**: 14B AWQ（約8.5GB）に最適。CU消費は約 1.5〜2.0 CU/h（月50〜65時間稼働可能）。
- **セーフティガードレール（検閲）の回避**: 商用APIで拒否されがちな攻撃コードやエクスプロイト検証を制約なく実行。
- **ゼロコンフィグ自動同期**: Colab 起動時は自動で Strixlibe のルーティング先になり、停止時は既存の **OpenCode (Zen)** 設定へ自動フォールバック。
- **CU保護ウォッチドッグ**: スキャン完了後、20分間無通信で `runtime.unassign()` によりインスタンスを自動切断。

---

## 🚀 Google Colab で直接開く

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ba-essa/colab-strix-llm/blob/main/colab_strix_llm_server.ipynb)

---

## 📋 選択可能モデル一覧（10分以内ロード保証）

1. `casperhansen/deepseek-r1-distill-qwen-32b-awq`（**既定値**: 約19GB、DL 3〜5分。32B 最高峰の推論思考、L4推奨）
2. `Qwen/Qwen2.5-Coder-32B-Instruct-AWQ`（約19GB、DL 3〜5分。32B コーディング特化、L4推奨）
3. `casperhansen/deepseek-r1-distill-qwen-14b-awq`（約8.5GB、DL 1〜2分！爆速ロード・推論思考、T4/L4推奨）
4. `Qwen/Qwen2.5-Coder-14B-Instruct-AWQ`（約8.5GB、DL 1〜2分！爆速ロード・コーディング特化、T4/L4推奨）
5. `Qwen/Qwen2.5-Coder-7B-Instruct-AWQ`（約5GB、DL 1分未満！最速・T4最安）
