# 学習ロードマップ

このプロジェクトで探求する技術領域の学習順序と内容を定義する。

**目的**: Context EngineeringやAI Agentの基礎となる低レイヤー技術を、実装・計測を通じて原理から理解する。

**基本方針**: 「実装して計測する」。読んで理解した気になるのではなく、動くコードと数字で確認する。

---

## スコープ

| | 内容 |
|---|---|
| **llm-labでやること** | Layer 1〜3（情報の表現・検索・コンテキスト構築）を自前で実装・計測 |
| **業務でやること** | Layer 4（エージェントの機構・Context Engineering・プロンプト戦略）を実践 |

---

## フェーズ構成

```
Layer 1: 情報の表現
  Phase 1: Embedding & 類似度指標
  Phase 2: Tokenization
  Phase 3: Sparse Retrieval（BM25）

Layer 2: 情報の検索
  Phase 4: ANN（近似最近傍探索）
  Phase 5: Hybrid Search & RRF
  Phase 6: Re-ranking

Layer 3: コンテキストの構築
  Phase 7: Chunking戦略
  Phase 8: Attentionの特性
  Phase 9: Token budget管理

Layer 4: エージェントの機構（業務で学ぶ・llm-labのスコープ外）
```

各フェーズは前フェーズの知識を前提とする。ただし探求の中で前フェーズに戻って深掘りすることを歓迎する。

---

## Layer 1: 情報の表現

### Phase 1: Embedding & 類似度指標

**Why**: Embeddingはすべての意味検索・メモリシステムの土台。距離指標の理解なしにRetrieval品質を語れない。

#### 距離・類似度指標

| 指標 | 計算方法 | 特徴・使いどころ |
|---|---|---|
| コサイン類似度 | ベクトルのなす角のcos値 | 長さを無視して方向のみ比較。テキストに最適 |
| ドット積（内積） | 各成分の積の和 | 長さも考慮。正規化済みembeddingならcosineと等価 |
| ユークリッド距離 | 2点間の直線距離 | 長さの差が意味を持つ場合に使う |

#### 評価指標

| 指標 | 概要 | 使いどころ |
|---|---|---|
| Recall@k | 正解文書のうちTop-k件に含まれる割合 | 「取りこぼし」を測る。最も基本的な指標 |
| MRR（Mean Reciprocal Rank） | 最初の正解文書の順位の逆数の平均 | 最も重要な1件がどこにあるかを測る |
| nDCG（Normalized DCG） | 関連度に段階スコアをつけて順位を考慮した指標 | 複数正解・部分的正解がある場合に適切 |

**実装タスク**
- NumPyでコサイン類似度・ドット積・ユークリッド距離を実装する
- 同一クエリセットでRecall@k・MRR・nDCGを計測・比較する
- OpenAI embeddingが正規化済みかを確認し、cosineとdot積の差を実測する

**探求ポイント**
- OpenAI embeddingはどの距離指標を前提として設計されているか
- pgvectorの `<=>` / `<#>` / `<->` 演算子はそれぞれ何に対応するか
- 正規化の有無が結果に与える影響
- Recall@k / MRR / nDCGはどんな場面で使い分けるか

---

### Phase 2: Tokenization

**Why**: LLMは文字ではなくトークンを処理する。Context Windowの管理・コスト計算・チャンキング設計のすべてに直結する。

#### 主要な概念

- **BPE（Byte Pair Encoding）**: 頻出する文字ペアを繰り返しマージして語彙を構築するトークン化方式
- **cl100k_base / o200k_base**: OpenAIが使うトークナイザー。モデルによって異なる
- **Token効率**: 同じテキストでも言語・モデルによってトークン数が大きく異なる

**実装タスク**
- tiktokenで日本語・英語の同一意味文のトークン数を比較する
- チャンクを文字数で区切った場合とトークン数で区切った場合のズレを計測する
- コンテキストウィンドウを埋めていくときのコスト増加を定量化する

**探求ポイント**
- 日本語は英語の何倍のトークンを消費するか
- 文字数ベースのチャンク境界とトークン境界のズレがどの程度問題になるか

---

### Phase 3: Sparse Retrieval（BM25）

**Why**: キーワード検索はEmbedding検索と得意・不得意が異なる。Hybrid Searchを理解するためにも必須。

#### BM25（Best Match 25）

TF-IDFの改良版。現在のキーワード検索の実質的な標準。

```
BM25(D, Q) = Σ IDF(qi) × [TF(qi,D) × (k1+1)] / [TF(qi,D) + k1×(1 - b + b×|D|/avgdl)]
```

- `TF`: 文書内の単語出現頻度（飽和関数で上限あり）
- `IDF`: 逆文書頻度（レアな単語ほど重要）
- `k1`: TFの飽和度（1.2〜2.0が典型値）
- `b`: 文書長の正規化強度（0〜1、0.75が典型値）

**TF-IDFとの違い**: TFに飽和関数を適用することで、同じ単語が大量に出現する文書を過大評価しない。

#### SPLADE（Sparse Lexical and Expansion Model）

BM25をneural化したモデル。BERTのMLM headを使ってクエリ・文書をsparse vectorに変換し、同時にterm expansionを行う。

- BM25と同様にsparse vectorなのでinverted indexが使える
- 同義語・関連語への展開がBM25より高い精度を実現する
- BM25の理解があれば「なぜSPLADEが強いか」を説明できる

**実装タスク**
- BM25をゼロから実装する（ライブラリ不使用）
- Phase 1と同じクエリセットでRecall@kを比較する
- k1・bパラメータを変えて精度への影響を計測する

**探求ポイント**
- 日本語テキストでの挙動（形態素解析との組み合わせ）
- Embeddingが得意なクエリタイプ vs BM25が得意なクエリタイプの違い

---

## Layer 2: 情報の検索

### Phase 4: ANN（近似最近傍探索）

**Why**: 実用的なベクトル検索は全探索（KNN）では遅すぎる。HNSWを理解することでpgvectorのインデックス設計が原理から分かる。

#### HNSW（Hierarchical Navigable Small World）

```
Layer 2:  A --------- F          (粗い層: 遠距離リンク)
Layer 1:  A --- C --- F --- H    (中間層)
Layer 0:  A-B-C-D-E-F-G-H-I    (密な層: 近距離リンク)
```

- 多層グラフ構造で近傍を階層的に探索
- 上位層から粗く絞り込み、下位層で精密に探す
- 完全な全探索（KNN）より大幅に高速。精度はわずかに低下

**pgvectorのHNSWパラメータ**

| パラメータ | 意味 | 精度 | メモリ | インデックス構築速度 |
|---|---|---|---|---|
| `m` | 各ノードの最大リンク数 | ↑ | ↑ | ↓ |
| `ef_construction` | 構築時の探索幅 | ↑ | - | ↓ |
| `ef_search` | 検索時の探索幅 | ↑ | - | - (クエリ速度↓) |

**実装タスク**
- hnswlibでHNSWインデックスを構築し、KNNとのRecall@k・レイテンシを比較する
- m / ef_construction / ef_searchを変化させて精度とレイテンシのトレードオフを実測する
- データ量を増やしながらKNNとHNSWのレイテンシ差がどう広がるかを計測する

**探求ポイント**
- KNNとHNSWの精度差はデータ量が増えるとどう変化するか
- IVFFlat（別のANNアルゴリズム）とのトレードオフ

---

### Phase 5: Hybrid Search & RRF

**Why**: DenseとSparseを統合することで、単体より高いRecall@kを実現できる。業務での即効性が高い。

#### RRF（Reciprocal Rank Fusion）

```
RRF_score(d) = Σ 1 / (k + rank_i(d))   (k=60が典型値)
```

スコアの絶対値ではなく順位を使うため、DenseとSparseのスコールスケールの違いを気にしなくてよい。

**実装タスク**
- Phase 1（Dense）とPhase 3（Sparse）の結果をRRFで統合する
- Dense単体 / Sparse単体 / Hybrid の3条件でRecall@kを比較する
- RRFのkパラメータの感度分析をする

**探求ポイント**
- どのクエリタイプでHybridが最も効くか
- kパラメータへの精度の感度

---

### Phase 6: Re-ranking

**Why**: 一次検索で取れなかった精度を後段で補う。コスト・レイテンシとのトレードオフが最も顕著なフェーズ。

#### Bi-encoder vs Cross-encoder

| | Bi-encoder（embedding） | Cross-encoder（re-ranker） |
|---|---|---|
| 処理方法 | クエリと文書を独立にエンコード | クエリと文書をペアでエンコード |
| 精度 | 中 | 高 |
| レイテンシ | 低（事前計算可能） | 高（全ペアを都度計算） |
| 用途 | 一次検索 | 再ランキング |

**典型的なパイプライン**

```
一次検索（Bi-encoder）: Top-100件を高速に取得
        ↓
Re-ranking（Cross-encoder）: Top-100件をスコア再計算
        ↓
最終結果: Top-5件をLLMに渡す
```

**実装タスク**
- Cross-encoderありなしでEnd-to-End精度を比較する
- Re-rankerが追加するレイテンシを計測する

**探求ポイント**
- Re-rankerの有無でRecall@kはどれくらい変わるか
- Cross-encoderのレイテンシはボトルネックになるか

---

## Layer 3: コンテキストの構築

### Phase 7: Chunking戦略

**Why**: Layer 1〜2の知識があると「なぜchunk sizeが精度に影響するか」が原理から説明できる。

| 戦略 | 概要 | 向いているケース |
|---|---|---|
| Fixed-size | 固定トークン数で分割 | シンプル、ベースライン |
| Sentence-based | 文境界で分割 | 意味のまとまりを保ちたい |
| Semantic Chunking | embedding類似度で境界を決定 | 話題の変化点で切りたい |
| Parent-Document Retrieval | 小chunkで検索、大chunkで回答生成 | 精度とコンテキスト量の両立 |
| Late Chunking | 文書全体をembeddingしてからchunk分割 | 文書全体の文脈をchunk embeddingに保持したい |
| Contextual Retrieval | LLMで各chunkにコンテキスト文を付加してからembedding/BM25 | 孤立したchunkの文脈欠落を補いたい（Anthropic, 2024） |

**実装タスク**
- Fixed-size / Sentence-based / Semantic / Parent-Document の4戦略を実装してRecall@kで比較する
- chunk_size / overlapのパラメータ感度分析をする
- Late Chunking vs 通常のChunkingの精度差を計測する
- Contextual Retrievalのコスト増加（LLM呼び出し）と精度改善のトレードオフを計測する

**探求ポイント**
- chunk_size変化がRecall@kに与える影響の非線形性
- 日本語文書でのSemantic Chunkingの品質
- Late ChunkingとContextual Retrievalはどんなクエリタイプで差が出るか

---

### Phase 8: Attentionの特性

**Why**: LLMがコンテキストのどこを重視するかを理解することが、Context Engineeringの設計判断の根拠になる。業務でContext Engineeringを実践する前に原理を知っておく。

#### 主要な概念

- **Lost in the Middle**: コンテキストの中間にある情報は無視されやすい（Liu et al., 2023）
- **Primacy / Recency bias**: 先頭と末尾の情報が処理されやすい傾向
- **コンテキスト長と精度の関係**: コンテキストが長くなるほど中間部の情報が使われにくくなる

**実装タスク**
- 正解情報をコンテキストの先頭・中間・末尾に配置し、回答精度を比較する
- コンテキスト長を変化させて精度低下のパターンを計測する

**探求ポイント**
- 何トークン以降から「中間」の影響が出始めるか
- モデルごとに特性は異なるか

---

### Phase 9: Token budget管理

**Why**: コンテキストに何を入れて何を捨てるかの判断が、精度・レイテンシ・コストの3軸すべてに影響する。Phase 8の知識が判断の根拠になる。

#### 主要な概念

Context Engineeringの操作は4つに分類できる。

| 操作 | 概要 |
|---|---|
| **Select** | 何をコンテキストに含めるかの選択（Re-rankスコア・トークン数での優先順位付け） |
| **Compress** | コンテキストのトークン数を削減（Context Compression・LLMLingua） |
| **Isolate** | コンテキストを役割別に分離（system prompt / user prompt / retrieved context の明確な区別） |
| **Write** | 外部ストアへの書き込み（長期メモリ・会話サマリーの保存）→ Layer 4で扱う |

**実装タスク**
- コンテキスト量（トークン数）と回答品質・コストのトレードオフを計測する
- Context Compressionありなしで精度・コスト・レイテンシを比較する
- system prompt / retrieved context / conversation historyを分離したときの回答品質変化を計測する

**探求ポイント**
- 「多ければ良い」ではないcontext量の閾値を実測する
- Compressionによる情報ロスはどの程度か
- Isolateの設計が回答の一貫性にどう影響するか

---

## Layer 4: エージェントの機構（業務で学ぶ）

llm-labのスコープ外。業務での実践を通じて習得する。

| 技術 | 概要 |
|---|---|
| Function calling / Structured output | LLMがツールを呼び出す仕組み |
| Memory / State管理 | エージェントが状態を保持・更新する仕組み |
| Planning（ReAct等） | 思考と行動を交互に繰り返すパターン |
| Multi-agent Orchestration | 複数エージェントの協調 |
| プロンプト戦略（HyDE・Multi-Query等） | クエリ変換・コンテキスト管理の実践 |

---

## 3軸最適化との対応表

| フェーズ | 精度 | レイテンシ | コスト |
|---|---|---|---|
| Phase 1: Embedding & 類似度 | ◎ | ○ | ○ |
| Phase 2: Tokenization | ○ | ○ | ◎ |
| Phase 3: BM25 | ○ | ○ | ○ |
| Phase 4: ANN | ○ | ◎ | ○ |
| Phase 5: Hybrid Search | ◎ | △ | ○ |
| Phase 6: Re-ranking | ◎ | △ | △ |
| Phase 7: Chunking | ◎ | ○ | ◎ |
| Phase 8: Attentionの特性 | ◎ | - | - |
| Phase 9: Token budget | ◎ | △ | ◎ |

◎: 直接的に大きく影響　○: 影響あり　△: 設計次第　-: 直接的な影響なし

---

## 実験設計の方針

各フェーズで「仮説 → 実装 → 計測 → 考察」のサイクルを回す。

```
仮説を立てる
    ↓
llm-labで実装
    ↓
Recall@k / MRR / nDCG / Faithfulness / レイテンシ / コスト を計測
    ↓
結果を考察・次の仮説へ
```
