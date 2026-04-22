# Windows 11 + WSL2(Ubuntu) で Robot Framework + Playwright を用いた検証自動化環境を構築する手順

## 1. 目的
Windows 11 上の WSL2(Ubuntu) 環境に、Robot Framework と Playwright ベースの Browser Library を導入し、ブラウザ自動テストを実行できる状態にする。

本手順では、以下の方針を採用する。

- **Windows 側**
  - VS Code をインストール済みとする
  - WSL2 を利用する
  - エディタ操作や WSL 接続を行う

- **WSL(Ubuntu) 側**
  - Python 仮想環境を作成する
  - Robot Framework / Browser Library を導入する
  - Playwright 実行に必要な Linux 依存ライブラリを導入する
  - テストを実行する

---

## 2. 構成方針
Browser Library は Playwright ベースの Robot Framework 用ライブラリである。  
Node.js を個別に管理しない簡易構成として、`robotframework-browser[bb]` を利用する。

この構成では以下を実施する。

1. Python 仮想環境作成
2. `robotframework` と `robotframework-browser[bb]` をインストール
3. `rfbrowser install` で Playwright ブラウザバイナリ導入
4. Linux 依存ライブラリ不足時は OS パッケージを追加
5. Robot テストを実行

---

## 3. 事前条件
- Windows 11
- WSL2 有効化済み
- Ubuntu 導入済み
- Windows 側に VS Code インストール済み
- インターネット接続可能

---

## 4. どちらで実行するか
### Windows 側で実行するもの
- VS Code 起動
- `WSL: Connect to WSL`
- 必要に応じて `wsl --update`

### WSL(Ubuntu) 側で実行するもの
- `apt` によるパッケージ導入
- `python3 -m venv`
- `pip install`
- `rfbrowser install`
- `robot -d results tests`

---

## 5. Windows 側の準備

### 5.1 WSL を最新化する（任意だが推奨）
PowerShell で実行する。

```powershell
wsl --status
wsl --update
wsl --shutdown
```

### 5.2 VS Code から WSL に接続する
1. Windows で VS Code を起動する
2. 拡張機能で **WSL** 拡張をインストールする
3. `Ctrl + Shift + P`
4. `WSL: Connect to WSL` を実行する

---

## 6. WSL(Ubuntu) 側のセットアップ

以下は **WSL の Ubuntu ターミナル** で実行する。

### 6.1 作業ディレクトリを作成する

```bash
mkdir -p ~/work/rf-playwright
cd ~/work/rf-playwright
```

VS Code をこのフォルダで開く場合:

```bash
code .
```

---

### 6.2 Python / venv / pip を導入する

```bash
python3 --version
sudo apt update
sudo apt install -y python3 python3-pip python3-venv
```

---

### 6.3 Python 仮想環境を作成する

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip setuptools wheel
```

---

## 7. Robot Framework / Browser Library の導入

### 7.1 Robot Framework と Browser Library を導入する

```bash
pip install robotframework robotframework-browser[bb]
```

---

### 7.2 Playwright ブラウザバイナリを導入する

```bash
rfbrowser install
```

`rfbrowser` が見つからない場合は以下を使う。

```bash
python -m Browser.entry install
```

---

## 8. Linux 側の依存ライブラリを導入する
WSL 上では Chromium 実行時に Linux の共有ライブラリ不足で失敗することがある。  
今回の環境でも `libnspr4.so` 不足が発生し、以下導入後にテストが成功した。

```bash
sudo apt update
sudo apt install -y \
  libnspr4 \
  libnss3 \
  libatk-bridge2.0-0 \
  libdrm2 \
  libxkbcommon0 \
  libgbm1 \
  libgtk-3-0
```

必要に応じて音声系ライブラリも追加する。

```bash
sudo apt install -y libasound2t64 || sudo apt install -y libasound2
```

---

## 9. Playwright 依存を自動導入する代替手順（任意）
OS 依存を広めに自動導入したい場合は、Playwright CLI を使う方法もある。

```bash
source .venv/bin/activate
pip install playwright
sudo .venv/bin/playwright install-deps chromium
```

必要に応じて Browser Library 側を再初期化する。

```bash
rfbrowser clean-node
rfbrowser install
```

---

## 10. 動作確認用テストを作成する

### 10.1 テストディレクトリ作成

```bash
mkdir -p tests
```

### 10.2 `tests/smoke.robot` を作成する

```bash
cat > tests/smoke.robot <<'EOF2'
*** Settings ***
Library    Browser

*** Test Cases ***
Open Example
    New Page    https://example.com
    Get Text    h1    contains    Example Domain
EOF2
```

---

## 11. テストを実行する

```bash
source .venv/bin/activate
robot -d results tests
```

成功時の想定出力例:

```text
==============================================================================
Tests
==============================================================================
Tests.Smoke
==============================================================================
Open Example                                                          | PASS |
------------------------------------------------------------------------------
Tests.Smoke                                                           | PASS |
1 test, 1 passed, 0 failed
==============================================================================
Tests                                                                 | PASS |
1 test, 1 passed, 0 failed
==============================================================================
Output:  /home/user/work/rf-playwright/results/output.xml
Log:     /home/user/work/rf-playwright/results/log.html
Report:  /home/user/work/rf-playwright/results/report.html
```

---

## 12. 成果物の確認

### 12.1 結果ファイル一覧を確認する

```bash
ls -l results
```

想定される主な出力:

- `results/output.xml`
- `results/log.html`
- `results/report.html`

### 12.2 XML の先頭を確認する

```bash
head /home/user/work/rf-playwright/results/output.xml
```

### 12.3 HTML レポートの先頭を確認する

```bash
head /home/user/work/rf-playwright/results/log.html
```

### 12.4 Windows のブラウザでレポートを開く

```bash
explorer.exe "$(wslpath -w results/log.html)"
explorer.exe "$(wslpath -w results/report.html)"
```

---

## 13. 今回の環境で確認できたこと
今回の実行結果から、以下が確認できている。

- Robot Framework が正常動作している
- Browser Library 経由で Playwright ブラウザ起動ができている
- `https://example.com` へのアクセスが成功している
- `h1` のテキスト検証が成功している
- `output.xml`, `log.html`, `report.html` が生成されている

ログ上では以下の内容も確認できる。

- `Robot 7.4.2`
- `Python 3.12.3 on linux`
- BrowserBatteries の gRPC server 起動
- 新規ブラウザ / context / page の自動生成

このため、**WSL2(Ubuntu) 上で Robot Framework + Playwright ベースの検証自動化環境が構築完了している** と判断してよい。

---

## 14. よくある失敗と対処

### 14.1 `rfbrowser: command not found`
```bash
python -m Browser.entry install
```

### 14.2 `libnspr4.so` など共有ライブラリ不足
以下を導入する。

```bash
sudo apt install -y \
  libnspr4 \
  libnss3 \
  libatk-bridge2.0-0 \
  libdrm2 \
  libxkbcommon0 \
  libgbm1 \
  libgtk-3-0
```

または Playwright 依存自動導入:

```bash
pip install playwright
sudo .venv/bin/playwright install-deps chromium
```

### 14.3 `results/` をそのままコマンド実行してしまう
`results/` はディレクトリ名であり、コマンドではない。  
確認したい場合は `ls -l results` を使う。

---

## 15. 最小実行コマンドまとめ

### 初回セットアップ
```bash
mkdir -p ~/work/rf-playwright
cd ~/work/rf-playwright

sudo apt update
sudo apt install -y python3 python3-pip python3-venv \
  libnspr4 libnss3 libatk-bridge2.0-0 libdrm2 libxkbcommon0 libgbm1 libgtk-3-0

python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U pip setuptools wheel
pip install robotframework robotframework-browser[bb]
rfbrowser install
```

### テスト作成
```bash
mkdir -p tests
cat > tests/smoke.robot <<'EOF2'
*** Settings ***
Library    Browser

*** Test Cases ***
Open Example
    New Page    https://example.com
    Get Text    h1    contains    Example Domain
EOF2
```

### テスト実行
```bash
source .venv/bin/activate
robot -d results tests
```

---

## 16. 補足
- 開発・実行主体は **WSL(Ubuntu) 側**
- VS Code は **Windows 側アプリ + WSL 接続**
- 今後テストケースを増やす場合は `tests/` 配下に `.robot` を追加していく
- CI/CD や再現性重視に進めるなら、次段階で Docker 化を検討するとよい
