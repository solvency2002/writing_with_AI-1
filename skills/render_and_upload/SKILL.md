---
name: render_and_upload
description: Render Markdown to Word and upload to Google Drive as a Google Doc.
---

# Render and Upload Skill

Markdown ファイルを Word (docx) にレンダリングし、Google ドライブに Google ドキュメント形式でアップロードする手順です。

## 前提条件

- `pandoc` がインストールされていること
- `rclone` がインストールされ、`gdrive:` リモートが設定されていること
- プロジェクト内に `demo/render_docx.ps1` が存在すること

## 実行手順

1. **レンダリング**:
   `demo/render_docx.ps1` を実行して、`draft.md` から `demo/output/draft.docx` を生成します。

2. **アップロード**:
   以下のコマンドを使用して、Google ドキュメント形式に変換しながらアップロードします。
   ```powershell
   rclone copy demo/output/draft.docx gdrive:writing_with_AI/ --drive-import-formats docx
   ```

3. **URL の取得**:
   アップロードされたファイルの ID を取得し、編集用 URL (`https://docs.google.com/document/d/<FILE_ID>/edit`) を作成して提示します。
   ID は以下のコマンドで確認できます。
   ```powershell
   rclone lsjson gdrive:writing_with_AI/draft.docx
   ```

## 注意事項

- Google ドキュメント形式に変換するため、`--drive-import-formats docx` フラグは必須です。
- 変換後のファイルは `rclone` 上でサイズが `-1` と表示されますが、これは正常な動作です。
