# 環境チェック (AIへの指示)

WS当日の冒頭で使う。IDEで本リポジトリを開いた状態で、AIチャットに次のように依頼する。

> `@env_check.md` を読んで、私のPC環境を順に確認してください。

AIはOS判定→必須ツール確認→不足分のインストール手順提案、までをこのファイルの指示に沿って実行する。

---

## AIへの依頼 (このセクションを順に実行)

### 1. OS判定

OSを判定し、結果を1行で宣言する。

- **Windows** の場合: `$PSVersionTable.OS` または `systeminfo | findstr /B /C:"OS Name"` を実行
- **macOS** の場合: `uname -sr` を実行
- それ以外 (Linux等) はサポート対象外として、Windows / macOS のどちらに近いか確認しつつユーザーに判断を仰ぐ

### 2. 必須ツールの確認

以下のコマンドを順に実行し、それぞれのバージョン (1行目で十分) を報告する。コマンドが見つからない場合は「未インストール」と報告する。

| 項目 | 確認コマンド | 必須度 |
|------|--------------|--------|
| pandoc | `pandoc --version` | **必須** |
| git | `git --version` | 推奨 |

### 3. 不足分のインストール提案

未インストールのツールがある場合のみ、下記「インストール手順」のうち該当OS・該当ツールの手順を **提示する**。**自動でインストールを実行しない** (参加者が読んで実行可否を判断する)。

すでに入っている場合は「OK」と短く返し、インストール手順は提示しない。

### 4. 結果サマリー

最後に、以下のフォーマットで結果を3行で報告する。

```text
OS: <Windows 11 | macOS 14.5 | ...>
pandoc: <2.x.x | 未インストール → 手順を上に提示>
git:    <2.x.x | 未インストール → 手順を上に提示>
```

---

## インストール手順

### Windows: pandoc

**winget が使える場合 (推奨)**:

```powershell
winget install --id JohnMacFarlane.Pandoc -e
```

**Chocolatey が使える場合**:

```powershell
choco install pandoc -y
```

**それ以外**: <https://pandoc.org/installing.html> から `pandoc-X.Y-windows-x86_64.msi` をダウンロードしてインストール。

確認 (PowerShellを再起動してから):

```powershell
pandoc --version
```

> `pandoc` がコマンドとして見つからない場合は、PowerShell を一度閉じて開き直す。新しいシェルを開き直さないと PATH が更新されないことがある。

### macOS: pandoc

**Homebrew が入っている場合 (推奨)**:

```bash
brew install pandoc
```

**Homebrew が入っていない場合**: まず Homebrew をインストール。

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

確認:

```bash
pandoc --version
```

### Windows: git

```powershell
winget install --id Git.Git -e
```

### macOS: git

`git --version` を初回実行すると Xcode Command Line Tools のインストールダイアログが出るので、そのまま進めれば入る。明示的に入れたい場合:

```bash
brew install git
```

---

## 制約 (AIへ)

- **削除、初期化、不要なOS設定変更は提案しない**: このファイルに記載のないコマンド (`rm -rf`、`Remove-Item -Recurse`、レジストリ変更、`sudo` 不要な操作の `sudo` 化など) は提案しない
- **このファイルに記載のないパッケージを追加で提案しない**: pandoc / git 以外のインストール提案 (LaTeX, Node, Python など) は不要。WS当日に必要なのは pandoc のみ
- **管理者権限が必要な操作は明示する**: `sudo` や「管理者として実行」が必要なら、その旨を一言添える
- **自動実行しない**: インストールコマンドはユーザーに提示するだけにとどめ、`pandoc --version` のような **確認系コマンド** のみ実行する
- **10分で `pandoc --version` が通らなければ深追いしない**: 個別環境の解決にこだわらず、講師画面のデモを追う案内に切り替える
