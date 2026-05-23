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
