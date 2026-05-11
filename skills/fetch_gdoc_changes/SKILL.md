---
name: fetch_gdoc_changes
description: Fetch tracked changes (suggestions) and comments from a Google Doc via rclone, without using a browser.
---

# Fetch Google Doc Track Changes & Comments via rclone

Google ドキュメントの「変更提案 (suggestion)」と「コメント」を、ブラウザを開かずに取得する手順。
DOCX としてエクスポートすると、現在オープン中の suggestion は Word のトラック変更 (`w:ins` / `w:del`) として、コメントは `comments.xml` として保持されるので、それをローカルでパースする。

## 前提条件

- `rclone` がインストール済みで、`gdrive:` リモートが設定されていること
- Python 3 が利用可能であること（標準ライブラリのみで動作）
- 対象 Google Doc のファイル ID（URL `https://docs.google.com/document/d/<FILE_ID>/edit` の `<FILE_ID>` 部分）

## 実行手順

### 1. DOCX としてエクスポート

ファイル ID を直接指定して落とす:

```bash
mkdir -p /c/tmp/gdoc_changes
rclone backend copyid gdrive: \
  <FILE_ID> \
  /c/tmp/gdoc_changes/ \
  --drive-export-formats docx
```

または、Drive 内のパスがわかっているなら:

```bash
rclone copy "gdrive:path/to/doc" /c/tmp/gdoc_changes/ \
  --drive-export-formats docx
```

エクスポートされた `<docname>.docx` には、開いている suggestion がトラック変更として、コメントも併せて含まれる。

### 2. DOCX をパースして変更とコメントを抽出

以下のスクリプトを `parse_docx_changes.py` として保存し、`python parse_docx_changes.py <docx_path>` で実行する。
`report.md` と `report.json` を同じディレクトリに書き出す。

```python
"""Extract w:ins / w:del tracked changes and comments from a DOCX."""
from __future__ import annotations
import json, sys, zipfile
from pathlib import Path
from xml.etree import ElementTree as ET

W = "{http://schemas.openxmlformats.org/wordprocessingml/2006/main}"

def gather_text(parent):
    out = []
    for el in parent.iter():
        if el.tag in (f"{W}t", f"{W}delText") and el.text:
            out.append(el.text)
        elif el.tag == f"{W}tab":
            out.append("\t")
        elif el.tag == f"{W}br":
            out.append("\n")
    return "".join(out)

def main(docx_path: Path):
    out_dir = docx_path.parent / (docx_path.stem + "_unzipped")
    out_dir.mkdir(exist_ok=True)
    with zipfile.ZipFile(docx_path) as z:
        z.extractall(out_dir)

    doc = ET.parse(out_dir / "word" / "document.xml").getroot()
    paragraphs = list(doc.iter(f"{W}p"))
    p_index = {id(p): i + 1 for i, p in enumerate(paragraphs)}

    changes = []
    for p in paragraphs:
        ctx = gather_text(p)
        for child in p.iter():
            if child.tag in (f"{W}ins", f"{W}del"):
                changes.append({
                    "kind": "insert" if child.tag == f"{W}ins" else "delete",
                    "author": child.attrib.get(f"{W}author", ""),
                    "date": child.attrib.get(f"{W}date", ""),
                    "text": gather_text(child),
                    "paragraph_index": p_index[id(p)],
                    "paragraph_context": ctx,
                })

    comments = []
    cpath = out_dir / "word" / "comments.xml"
    if cpath.exists():
        for c in ET.parse(cpath).getroot().findall(f"{W}comment"):
            comments.append({
                "id": c.attrib.get(f"{W}id", ""),
                "author": c.attrib.get(f"{W}author", ""),
                "date": c.attrib.get(f"{W}date", ""),
                "text": gather_text(c),
            })

    # commentRangeStart/End でアンカーテキストを拾う
    flat = list(doc.iter())
    anchors = {}
    for i, el in enumerate(flat):
        if el.tag == f"{W}commentRangeStart":
            cid = el.attrib.get(f"{W}id", "")
            for j in range(i + 1, len(flat)):
                e = flat[j]
                if e.tag == f"{W}commentRangeEnd" and e.attrib.get(f"{W}id") == cid:
                    anchors[cid] = "".join(
                        (x.text or "") for x in flat[i:j] if x.tag == f"{W}t"
                    )
                    break
    for c in comments:
        c["anchor_text"] = anchors.get(c["id"], "")

    report = {"tracked_changes": changes, "comments": comments}
    (docx_path.parent / "report.json").write_text(
        json.dumps(report, ensure_ascii=False, indent=2), encoding="utf-8"
    )

    md = [f"# {docx_path.name} — track changes / comments\n",
          f"- 変更追跡: {len(changes)} 件",
          f"- コメント: {len(comments)} 件\n",
          "## トラック変更\n"]
    for i, c in enumerate(changes, 1):
        sign = "＋" if c["kind"] == "insert" else "−"
        md += [f"### {i}. {sign} {c['kind']} — {c['author']} @ {c['date']}",
               f"段落 #{c['paragraph_index']}", "",
               "**変更テキスト:**", "```", c["text"], "```",
               "**段落コンテキスト:**", "```", c["paragraph_context"], "```", ""]
    md.append("## コメント\n")
    for c in comments:
        md += [f"### コメント {c['id']} — {c['author']} @ {c['date']}",
               f"**アンカー:** `{c['anchor_text']}`", "", c["text"], ""]
    (docx_path.parent / "report.md").write_text("\n".join(md), encoding="utf-8")
    print(f"changes={len(changes)} comments={len(comments)}")

if __name__ == "__main__":
    main(Path(sys.argv[1]))
```

実行例:

```bash
python parse_docx_changes.py /c/tmp/gdoc_changes/draft.docx
```

## 取得できるもの / 取得できないもの

| 対象 | rclone + DOCX で取れる? | 補足 |
|---|---|---|
| 現在オープン中の suggestion（trackあり編集） | ✅ | `w:ins` / `w:del` |
| 現在のコメント本文・著者・日時 | ✅ | `comments.xml` |
| コメントのアンカーテキスト | ✅ | `commentRangeStart` / `commentRangeEnd` |
| Accept/Reject 済を含む全リビジョン履歴 | ❌ | Drive `revisions.list` API が必要 |
| コメントの返信スレッド構造 | △ | DOCX には返信もまとまって入るが構造は浅い。完全な木構造は Drive `comments`/`replies` API |

## 注意事項

- `--drive-export-formats docx` は **エクスポート** 用のフラグ（Google Doc → DOCX）。アップロード時の `--drive-import-formats docx` とは別物なので混同しない。
- Google Docs では「すでに accept した変更」は track changes として残らないため、DOCX 経由では取得できない。レビュー前の生の suggestion を保全したいなら、レビュアーが accept する前に取得する運用にする。
- 出力 `report.md` をそのままレビューコメントの一覧として AI への入力に使える。
