# Claude / AI Assistant Notes

このリポジトリでは、実際に書く原稿や個別案件の作業ファイルは `projects/` の下位フォルダに置く。

## `projects/` の扱い

- `projects/` 配下は、参加者・研究者ごとの実原稿、未公開データ、共同研究メモを置く作業領域。
- `projects/*` は `.gitignore` によりデフォルトで Git 管理外。
- サンプルとして共有する教材は `demo/` に置き、実案件の原稿やデータは `demo/` には置かない。
- `projects/` 配下のファイルを Git に追加する必要がある場合は、内容に個人情報、未公開データ、共同研究上の機密が含まれないことを確認してから、明示的に `.gitignore` の例外を追加する。

AI アシスタントは、ユーザーから明示された場合を除き、`projects/` 配下の実原稿やデータを Git 管理対象にしない。

## ケースレポートを書くとき

ケースレポート (症例報告) を最初から最後まで AI と一緒に書く場合は、`case_report_workflow` Skill (`skills/case_report_workflow/SKILL.md`) を起点にする。個別の作業 (CARE チェックだけ / 類似症例検索だけ など) は、対応する単体 Skill を直接呼ぶ。

## Letter to the Editor を書くとき

既発表論文への応答・批評として Letter to the Editor を AI と一緒に書く場合は、`letter_to_editor` Skill (`skills/letter_to_editor/SKILL.md`) を起点にする。日本語メモを入力に、`projects/<name>/` に英文レターのドラフトを生成する。レターは「(P1) 称賛 → (P2)(P3) 結論をゆがめるバイアス / 結果解釈の spin を指摘」の固定 3 段落構成。

- 対象論文ファイルは**著者が手で `projects/<name>/` に置く**。Skill は Read するだけで Web からは取得しない。
- 着手前に、対象論文の同定情報と批評対象の引用箇所を提示してユーザーと確認する human-in-the-loop ゲートを通す。
- 語数・文献数制限はメモ記載があればそれを使い、なければユーザーに確認する (Skill 側でガイドラインは取得しない)。
- 自験データ・症例を含むレターはこの Skill の対象外。その場合は `case_report_workflow` を使う。

個別の作業 (生成済みレターの査読だけ など) は、対応する単体 Skill を直接呼ぶ。レター固有の査読は `letter_review_simulator` Skill (`skills/letter_review_simulator/SKILL.md`)。

## PubMed 文献検索 (検索式は projects 配下に保存)

レターやケースレポートの文献検索は、コミット済みの汎用スクリプト `skills/similar_cases_search/scripts/pubmed_search.py` を使う。Web 検索ツールを持たないモデル (例: 検索できない Gemini) に作業を渡しても、このスクリプトを実行すれば PubMed から根拠付き BibTeX を得られる。

- **検索式 (query) はリポジトリに置かず、gitignore された `projects/<name>/searches/<topic>.query.txt` に保存**してから使う。リポジトリにはエンジンだけを置き、案件固有の検索内容で汚さない。
- `search` サブコマンドはレビュー用ステージング `searches/<topic>.candidates.md` を出力する (これは `.bib` ではない)。`add` サブコマンドだけが、著者承認済み PMID をレンダリング正本の `refs.bib` に追記する。野良 `.bib` を作らないので、pandoc レンダリング時の取り違えを防ぐ。
- NCBI コンタクトメールはハードコードしない。`--email` か環境変数 `$NCBI_EMAIL` で渡す。
- 使い方の詳細は `skills/similar_cases_search/scripts/README.md`。
