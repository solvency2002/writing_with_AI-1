---
name: render_and_upload
description: Render a Markdown manuscript to Word (docx) with pandoc and upload it to Google Drive as a Google Doc.
---

# Render and Upload Skill

指定された Markdown ファイルを Word (docx) にレンダリングし、Google ドライブに Google ドキュメント形式でアップロードする手順です。

## 前提条件

- `pandoc` がインストールされていること
- `rclone` がインストールされ、`gdrive:` リモートが設定されていること

## 入力

- **必須**: レンダリング対象の Markdown パス (例: `projects/<name>/draft.md`)。
  指定がなければユーザーに確認する。`demo/draft.md` はワークショップ練習用。
- **任意**: `refs.bib` と CSL ファイル。入力と同じフォルダ (または
  `demo/styles/`) にあれば `--citeproc` 付きでレンダリングする。
- **任意**: アップロード先フォルダ (デフォルト:
  `gdrive:writing_with_AI/<プロジェクト名>/`)。

## 実行手順

1. **レンダリング**:
   引用がある原稿は bibliography + CSL を付けてレンダリングする。

   ```powershell
   pandoc <input>.md --citeproc --bibliography <refs.bib> --csl <style>.csl -o <output_dir>/<input>.docx
   ```

   - 出力 docx は入力と同じプロジェクトフォルダ配下 (例:
     `projects/<name>/output/`) に置く。**実案件の成果物を `demo/output/`
     に置かない** (`demo/` は教材、`projects/` は実案件 — リポジトリの
     CLAUDE.md 参照)。
   - ワークショップ練習 (`demo/draft.md`) では `demo/render_docx.ps1` を
     そのまま実行してよい (出力は `demo/output/draft.docx`)。

2. **アップロード**:
   Google ドキュメント形式に変換しながらアップロードする。

   ```powershell
   rclone copy <output_dir>/<docname>.docx gdrive:writing_with_AI/<subfolder>/ --drive-import-formats docx
   ```

3. **URL の取得**:
   アップロードされたファイルの ID を取得し、編集用 URL
   (`https://docs.google.com/document/d/<FILE_ID>/edit`) を作成して提示します。

   ```powershell
   rclone lsjson gdrive:writing_with_AI/<subfolder>/<docname>.docx
   ```

## 注意事項

- Google ドキュメント形式に変換するため、`--drive-import-formats docx` フラグは必須です。
- 変換後のファイルは `rclone` 上でサイズが `-1` と表示されますが、これは正常な動作です。
- `projects/` 配下の実原稿 (未投稿原稿・個人情報を含み得る) をアップロードする場合は、アップロード先 Drive フォルダの共有範囲を著者が確認してから実行してください。
