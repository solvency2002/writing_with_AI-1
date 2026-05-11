# Writing with AI プロンプト集

## まず読ませる

```text
@demo/draft.md を読んでください。
今日はこのファイルを、臨床疫学の短い原稿として整えます。
まだ本文は書き換えず、構成上の問題と、引用が必要そうな箇所だけを箇条書きで指摘してください。
```

## 部分修正

```text
@demo/draft.md の Background セクションだけを対象にしてください。

方針:
- 内容は増やしすぎない
- 研究計画書として自然な日本語にする
- 主張を強くしすぎない
- 修正後の本文と、主な修正点を出してください
```

## 引用制限

```text
@demo/draft.md と @demo/refs.bib を読んでください。
本文中の引用は @demo/refs.bib にある文献だけを使ってください。
refs.bib にない文献名、DOI、PMIDは作らないでください。
引用を追加すべき場所を提案してください。
```

## 文体調整

```text
@demo/draft.md と @skills/humanizer_academic/SKILL.md を読んでください。
Discussion セクションだけを、SKILL.md の方針に沿って、学術的だが直訳調ではない日本語に直してください。
```

## pandocコマンドを作らせる

```text
@demo/draft.md @demo/refs.bib @demo/styles/american-medical-association.csl を使って、
Markdownをdocxに変換するpandocコマンドをPowerShell用に作ってください。
出力先は demo/output/draft.docx にしてください。
```

## 図を作らせる

```text
@demo/assets/study_flow.mmd を参考に、研究の流れを示すMermaid図を改善してください。
あとでdraw.ioで手直しできるように、必要ならdraw.io XML版も出してください。
```

## 参考: refs.bib に文献を入れる手順

AIに文献を捏造させないために、`refs.bib` の中身は reference manager から BibTeX をコピペして埋める。

- **Zotero** / **Paperpile** で論文を集める → 各エントリーから BibTeX をコピー → `demo/refs.bib` に貼る
- **DOI / PMID から直接 BibTeX を取りたい場合**:
  - <https://doi.org/> や <https://www.crossref.org/> で DOI 検索 → "Cite" から BibTeX 出力
  - PubMed なら NBIB → BibTeX 変換ツールを通す
- **Paperpile / Zotero と GitHub を自動連携する場合**:
  - 参考手順 (ChatGPT 共有): <https://chatgpt.com/share/69d1b863-35e8-83a7-9848-d1d84d8c8254>
  - reference manager の export を GitHub Actions などで `refs.bib` に同期する流れ

引用キーは reference manager 既定 (例: `kataoka2020`, `Youe085863`) のまま使うと安定。本文では `[@key1; @key2]` のように `;` 区切りでつなぐ（スペースは入れない）。

