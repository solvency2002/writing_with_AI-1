# Writing with AI — 参加者用リポジトリ

SRWS-PSG ハンズオンカフェ「Writing with AI」(2026-05-13 17:30-18:30) で使う最小ファイル一式。

## このリポジトリの位置づけ

「文章作成も、コード作成と同じく **プロジェクトをIDEで開き、ファイルをAIに見せながら、小さく直して、レンダリングして確認する** 作業にできる」を体験するためのデモ素材。

前提:

- 前提資料「もうコードは書かない！臨床疫学のためのVibe Coding完全ガイド」を実施済み
- Antigravity をインストール済みで、AIチャットが動く

## 当日までにやること

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
2. `prompts/writing_prompts.md` のプロンプトを見ながら、AIに部分修正を依頼する
3. `demo/protocol.md` + `demo/analysis.R` を `@` で読ませて、`draft.md` の Methods のたたき台を AI に書かせる
4. `demo/refs.bib` にある文献だけを使って引用を入れる
5. `demo/assets/figure1.png` を本文に埋め込む
6. pandoc で `demo/output/draft.docx` を作る:

   ```powershell
   cd demo
   pandoc draft.md --citeproc --bibliography refs.bib --csl styles/american-medical-association.csl -o output/draft.docx
   ```

   または:

   ```powershell
   powershell -ExecutionPolicy Bypass -File demo/render_docx.ps1
   ```

7. 生成した `draft.docx` を Google Docs にアップロード→共有

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
├── prompts/
│   └── writing_prompts.md       # 当日コピペするプロンプト集
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
