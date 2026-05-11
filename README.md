# Writing with AI — 参加者用リポジトリ

SRWS-PSG ハンズオンカフェ「Writing with AI」(2026-05-13 17:30-18:30) で使う最小ファイル一式。

## このリポジトリの位置づけ

「文章作成も、コード作成と同じく **プロジェクトをIDEで開き、ファイルをAIに見せながら、小さく直して、レンダリングして確認する** 作業にできる」を体験するためのデモ素材。

前提:

- 前提資料「もうコードは書かない！臨床疫学のためのVibe Coding完全ガイド」を実施済み
- Antigravity をインストール済みで、AIチャットが動く

## 当日やること

1. このリポジトリを `git clone` または ZIP ダウンロードでローカルに置く

   ```powershell
   git clone https://github.com/youkiti/writing_with_AI.git
   ```

2. IDE (Antigravity / VS Code 等) でフォルダごと開く
3. `env_check.md` を開き、AIチャットに次のように頼む

   ```text
   @env_check.md を読んで、私のPC環境を順に確認してください。
   ```

   pandoc が未インストールなら、AIに案内された手順に沿って入れる（**pandoc 関連のコマンドだけ許可**。それ以外は拒否）。

## 当日の流れ

1. `demo/draft.md` をIDEで開く
2. 下の「コピペ用プロンプト集」のプロンプトを見ながら、AIに部分修正を依頼する
3. `demo/protocol.md` + `demo/analysis.R` を `@` で読ませて、`draft.md` の Methods のたたき台を AI に書かせる
4. `demo/refs.bib` にある文献だけを使って引用を入れる
5. `demo/assets/figure1.png` を本文に埋め込む
6. pandoc で `demo/output/draft.docx` を作る（下の「pandoc コマンド」をコピペ）
7. 生成した `draft.docx` を Google Docs にアップロード→共有

---

## コピペ用プロンプト集

> 参加者が当日コピペする内容はすべてここに集約しています。AIチャットにそのまま貼ってください。
> ファイルパスは「リポジトリのルートをIDEで開いた状態」を想定 (`@demo/draft.md` のように書く)。

### 0. まず環境を確認

```text
@env_check.md を読んで、私のPC環境を順に確認してください。
```

### 1. まず読ませる（書き換えさせない）

```text
@demo/draft.md を読んでください。
今日はこのファイルを、臨床疫学の短い原稿として整えます。
まだ本文は書き換えず、構成上の問題と、引用が必要そうな箇所だけを箇条書きで指摘してください。
```

### 2. 部分修正の基本形

```text
@demo/draft.md を読んでください。
Background セクションだけを対象に、以下の方針で修正してください。
・内容は増やさない
・300 words 以内
・臨床疫学の研究計画書として自然な文体
・修正案だけでなく、どこを変えたかも短く説明
```

### 3. 部分修正のバリエーション（用途別）

```text
@demo/draft.md の Methods だけ、PECO が明確になるように直してください。
```

```text
@demo/draft.md の Discussion だけ、言い過ぎを弱めてください。
```

```text
@demo/draft.md 全体を読んで、論理の飛びだけ指摘してください。本文は書き換えないでください。
```

### 4. 引用を `refs.bib` の中だけに制限する

```text
@demo/draft.md と @demo/refs.bib を読んでください。
本文中の引用は @demo/refs.bib にある文献だけを使ってください。
refs.bib にない文献名、DOI、PMID は作らないでください。
引用を追加すべき場所を提案してください。
```

### 5. 文体調整（Skill を併用）

```text
@demo/draft.md と @skills/humanizer_academic/SKILL.md を読んでください。
Discussion セクションだけを、SKILL.md の方針に沿って、学術的だが直訳調ではない日本語に直してください。
```

### 6. Methods を protocol + analysis から「翻訳」させる

```text
@demo/protocol.md と @demo/analysis.R を読んでください。
これをもとに、@demo/draft.md の Methods セクションのたたき台を書いてください。
・PECO が読み手にすぐ分かる構成
・解析手順は analysis.R に書かれている範囲だけを反映
・引用が必要な箇所は [@key] のプレースホルダで残す
```

### 7. 図を作らせる

```text
@demo/assets/study_flow.mmd を参考に、研究の流れを示す Mermaid 図を改善してください。
あとで draw.io で手直しできるように、必要なら draw.io XML 版も出してください。
```

### 8. pandoc コマンドを作らせる（または自分でコピペ）

AIに作らせる場合:

```text
@demo/draft.md @demo/refs.bib @demo/styles/american-medical-association.csl を使って、
Markdown を docx に変換する pandoc コマンドを PowerShell 用に作ってください。
出力先は demo/output/draft.docx にしてください。
```

そのまま使う場合（ルートから実行）:

```powershell
cd demo
pandoc draft.md --citeproc --bibliography refs.bib --csl styles/american-medical-association.csl -o output/draft.docx
```

スクリプト経由（ルートから実行）:

```powershell
powershell -ExecutionPolicy Bypass -File demo/render_docx.ps1
```

---

## フォルダ構成

```text
.
├── README.md                    # このファイル
├── env_check.md                 # 環境チェック (AIに読ませて使う)
├── install_pandoc.md            # pandoc インストールメモ
├── demo/                        # ハンズオンの最小プロジェクト
│   ├── README.md
│   ├── draft.md                 # 本文 (Markdown)
│   ├── protocol.md              # 研究計画書サンプル (Methods生成用)
│   ├── analysis.R               # 解析コードサンプル (Methods生成用)
│   ├── refs.bib                 # 引用に使ってよい文献リスト
│   ├── render_docx.ps1          # pandoc 実行スクリプト
│   ├── assets/                  # 図 (PNG, Mermaid, HTML, draw.io XML)
│   │   ├── figure1.png
│   │   ├── study_flow.mmd
│   │   ├── study_flow.html
│   │   └── study_flow.drawio
│   ├── styles/
│   │   └── american-medical-association.csl  # 引用スタイル
│   └── output/
│       └── draft.docx           # pandocで生成済みの完成見本
└── skills/
    └── humanizer_academic/
        └── SKILL.md             # 文体調整デモ用 Skill
```

## ボトムライン

- **Markdown が真実のソース**、`docx` はビルド成果物。Word 保存ではなく `pandoc` で都度生成する
- **AIには `refs.bib` の中の文献だけを使わせる**。hallu-citation を防ぐ
- **`@` でファイル指定して部分修正**。本文全体を一度に書き換えさせない
- **Methods は `protocol.md` + `analysis.R` から「翻訳」できる**。AI の活躍場面

## 関連

- 前提WS: もうコードは書かない！臨床疫学のための Vibe Coding 完全ガイド
- 公式ドキュメント: [pandoc](https://pandoc.org/), [Mermaid](https://mermaid.js.org/), [draw.io](https://www.drawio.com/)
- 引用スタイル: [Citation Style Language (CSL)](https://citationstyles.org/)
