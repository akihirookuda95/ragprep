# セッションコンテキスト：2026-02-15

## 1. 目的（Goal）

- Context Engineering・AI Agentの基礎となる低レイヤー技術を実装・計測で学ぶ実験場としてllm-labを位置づける
- `docs/learning-roadmap.md` をその目的に合わせて再設計する
- Phase 1（Embedding & 類似度指標）から実装を開始する準備を整える

---

## 2. 現在地（Current status）

**完了したこと**
- `docs/design.md`・`docs/overview.md` は削除済み（ユーザーが削除）
- `docs/learning-roadmap.md` を全面書き直し（4層9フェーズ構成）
- Web検索による不足点の洗い出しと追加修正

**未着手**
- プロジェクトの実装（コードなし）
- `CLAUDE.md` が存在しない（セッション中に作成したが消えている）

---

## 3. 重要な決定（Key decisions）

- **llm-labのスコープをLayer 1〜3に限定する**
  - 結論：Phase 1〜9（情報の表現・検索・コンテキスト構築）をllm-labで実装計測。Layer 4（AI Agent・Context Engineering実践・プロンプト戦略）は業務で学ぶ
  - 理由：業務では低レイヤーを実装する機会がない。低レイヤーの原理理解が業務でのフレームワーク活用・AI指示精度を高める
  - 記録先：`docs/learning-roadmap.md` 冒頭スコープ表

- **「プロダクトを作る過程で学ぶ」方針を採用**
  - 結論：llm-lab自体が「各技術を実装し前の手法と数字で比較できるフレームワーク」というプロダクト
  - 理由：制約（動かす・計測する）があることで理解が定着する。ユーザーの経験則とも一致
  - 記録先：なし（セッション内決定）

- **技術スタックはTypeScript / Bun / Vitest**
  - 結論：型安全・高速実行・TypeScript nativeサポート
  - 記録先：`CLAUDE.md`（再作成が必要）

---

## 4. 未決事項・不明点（Open questions / Unknowns）

- **Phase 1の実装をどのデータセットで行うか**
  - なぜ重要か：Recall@k / MRR / nDCG の計測にはクエリセットと正解データが必要
  - 何が分かれば決められるか：使用する文書の種類（日本語技術文書か英語か）を決める

- **`CLAUDE.md` の再作成**
  - なぜ重要か：次回セッションでClaudeが技術スタック・方針を把握するために必要
  - 何が分かれば決められるか：実装フェーズが始まればコマンドが確定する

---

## 5. 実装・アーキテクチャの要点（Architecture / Implementation notes）

**評価パイプライン（Phase 1〜3で構築）**

```
クエリセット + 正解データ
    ↓
Embedding生成（OpenAI API）
    ↓
類似度計算（cosine / dot / euclidean）
    ↓
Recall@k / MRR / nDCG を計測・比較
```

**設計原則**
- 各PhaseはインターフェースをそろえてRecall@kで横比較できる構造にする
- 実験結果はJSON Linesで追記保存（再現性確保）
- 外部API（OpenAI）はモック化してテスト可能にする

**技術スタック**：TypeScript / Bun / YAML+Zod / JSON Lines / OpenAI API / Vitest

---

## 6. 関連ファイル（Files touched / relevant files）

| ファイル | 変更内容 |
|---|---|
| `docs/learning-roadmap.md` | 全面書き直し。4層9フェーズ構成に再設計。Tokenization・Attention特性・Token budget・Late Chunking・Contextual Retrieval・SPLADE・nDCGを追加 |
| `docs/design.md` | ユーザーが削除（旧: チャンク戦略中心の設計書） |
| `docs/overview.md` | ユーザーが削除 |
| `CLAUDE.md` | 作成したが消えている。Phase 1実装開始前に再作成が必要 |

---

## 7. 評価文脈（Evaluation context）

**予定指標**

| 指標 | 概要 | 使いどころ |
|---|---|---|
| Recall@k | 正解文書のうちTop-k件に含まれる割合 | 基本指標。取りこぼし測定 |
| MRR | 最初の正解文書の順位の逆数の平均 | 重要な1件の順位を評価 |
| nDCG | 段階スコアつき順位評価 | 複数正解・部分的正解がある場合 |
| Faithfulness | 回答が取得文書に基づいているか（ハルシネーション率） | Phase 8以降 |

**注意点**
- データセットはまだ未定。Phase 1開始前に確定が必要
- 評価は各フェーズで「前フェーズとの差」を比較する設計

---

## 8. 次回やること（Next steps）

1. `CLAUDE.md` を再作成する（技術スタック・コマンド・品質方針）
2. プロジェクト構造を設計する（`src/`・`experiments/`・`data/` のディレクトリ設計）
3. 技術スタックをセットアップする（`bun init`・TypeScript設定・ESLint・Vitest）
4. Phase 1用のテストデータセットを決める（クエリセット + 正解文書）
5. Phase 1実装：NumPyでcosine / dot / euclidean を実装する
6. Phase 1計測：同一クエリセットでRecall@k・MRR・nDCGを比較する
7. Phase 1考察：「OpenAI embeddingはどの指標前提か」を数字で確認する

---

## 9. リスク（Risks / gotchas）

- **OpenAI APIコスト**: Embedding生成を大量に回すとコストが積み上がる。テスト用データは小規模（100文書以下）に抑える
- **日本語のtokenization**: 日本語テキストでは文字数とトークン数のズレが大きい。Phase 2で定量化するまではトークン数ベースで管理する
- **評価データの品質**: 正解データセットの品質が低いとRecall@kが無意味になる。Phase 1開始前にデータセット設計を丁寧に行う
- **実装の散漫化**: Phaseを飛ばして進めると「前フェーズの知識が前提」という積み上げ構造が崩れる。Phase 1→2→3の順番を守る
- **CLAUDE.mdの欠如**: 次セッションで技術スタックや方針がリセットされる。次回作業開始時に最初に再作成する

---

## 10. 参考（References）

- [Anthropic: Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — Contextual Retrieval: chunkにLLMで文脈付加。top-20失敗率を49%削減
- [arXiv: Late Chunking (2409.04701)](https://arxiv.org/abs/2409.04701) — 全文embeddingしてからchunk分割する手法
- [arXiv: SPLADE (2107.05720)](https://arxiv.org/abs/2107.05720) — BM25をneural化したsparse retrieval手法
- [LangChain: Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) — write/select/compress/isolateの4操作フレームワーク
- [Liu et al., 2023: Lost in the Middle](https://arxiv.org/abs/2307.03172) — LLMがコンテキスト中間の情報を無視しやすいことを実証
