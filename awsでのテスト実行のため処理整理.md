- 通信方法コマンド整理
- アップロード方法，ファイル差分のみ
- [NEW] テスト実行をリモートに入らず実施
- ダウンロード方法
	- リモートに入らず=local sshからコマンド接続するシェル
-  [NEW] 上記3点処理ワンライナー


## 通信方法コマンド整理

- 前提 : powershellで実行
	- 理由：wsl経由だとaws cli, ssm, ssh key setting面倒だから

## アップロード方法，ファイル差分のみ
### 方針
- 現状記載のscpではなく，rsyncを使用
### まとめてアップロードするコマンド
```
rsync -avz .\common/ .\utils/ pyproject.toml uv.lock .env pytest.ini Makefile conftest.py generated_testcode/conftest.py "generated_testcode/【仕様書パス】" psq-dev-ec2:/home/patesq/test-automation/
```
### 自動アップロードのpowershellスクリプト(upload_code.ps1)
```ps1
<#
.SYNOPSIS
    SSM/SSH経由でテスト自動化コード一式をEC2に差分アップロードします。
.EXAMPLE
    .\deploy.ps1 -SpecPath "auth/login_spec.xlsx"
#>
param (
    [Parameter(Mandatory=$true, HelpMessage="generated_testcode/ 内の仕様書パス（例: auth/login_spec.xlsx）を指定してください。")]
    [string]$SpecPath
)

# エラーが発生したら処理を中断する
$ErrorActionPreference = "Stop"

# リモートの宛先設定
$RemoteHost = "psq-dev-ec2"
$RemoteDir  = "/home/patesq/test-automation" # 末尾のスラッシュはなしに統一

# 1. 共通ファイル・ディレクトリの送信リスト
$CommonTargetList = @(
    "common",
    "utils",
    "pyproject.toml",
    "uv.lock",
    ".env",
    "pytest.ini",
    "Makefile",
    "conftest.py",
    "generated_testcode/conftest.py"
)

# 2. 引数の仕様書パス（フルローカルパスの構築）
# Windowsのバックスラッシュをrsync（Linux互換）が認識しやすいよう、スラッシュ「/」に置換します
$CleanSpecPath = $SpecPath.Replace('\', '/')
$FullLocalSpecPath = "generated_testcode/$CleanSpecPath"

Write-Host "=========================================" -ForegroundColor Cyan
Write-Host "🚀 差分アップロードを開始します..." -ForegroundColor Cyan
Write-Host "📄 対象仕様書: $FullLocalSpecPath" -ForegroundColor Yellow
Write-Host "=========================================" -ForegroundColor Cyan

# 存在チェック（共通ファイル）
foreach ($item in $CommonTargetList) {
    if (-not (Test-Path $item)) { Write-Error "❌ 送信対象が見つかりません: $item" }
}
# 存在チェック（仕様書ファイル）
if (-not (Test-Path $FullLocalSpecPath)) {
    Write-Error "❌ 仕様書ファイルが見つかりません: $FullLocalSpecPath"
}

# --------------------------------------------------
# rsync 実行パート
# --------------------------------------------------

# ① 共通ファイル・ディレクトリをまとめて転送
Write-Host "📦 共通設定・ソースコードを同期中..." -ForegroundColor Gray
rsync -avz $CommonTargetList "${RemoteHost}:${RemoteDir}/"

# ② 仕様書ファイルを、リモート側にも同じ階層構造（ディレクトリ）を作って転送
# --relative (-R) オプションを使うことで、引数に指定したパスの階層がそのままリモートに再現されます
Write-Host "📂 仕様書ファイルを階層維持して同期中..." -ForegroundColor Gray
rsync -avz -R $FullLocalSpecPath "${RemoteHost}:${RemoteDir}/"

Write-Host "=========================================" -ForegroundColor Green
Write-Host "✨ アップロードが正常に完了しました！" -ForegroundColor Green
Write-Host "=========================================" -ForegroundColor Green

```

```
.\upload_code.ps1 -SpecPath "01.検索/2.プロフェッショナル検索"

```

### 補足：rsyncのインストール手順
powershellだと未インストールの可能性有り，以下のインストール方法を参照

chocolatery入っている前提，前に入れてた気もする，なければいれる
```
choco install rsync -y
```

### 補足：chocolateryのインストール手順
```
Set-ExecutionPolicy Bypass -Scope Process -Force

[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org'))


choco --version
```



## テスト実行をリモートに入らず実施


```
<#
.SYNOPSIS
    SSM/SSH経由でEC2上のpytestを遠隔実行します（ファイル・ディレクトリ両対応）。
.EXAMPLE
    # 特定の仕様書ファイルを指定する場合
    .\run-test.ps1 -SpecPath "auth/login_spec.xlsx"
    
    # ディレクトリ配下のテストをまとめて実行する場合
    .\run-test.ps1 -SpecPath "auth/"
#>
param (
    [Parameter(Mandatory=\$true, HelpMessage="テスト対象の仕様書パス、またはディレクトリを指定してください。")]
    [string]\$SpecPath
)

# エラーが発生したら処理を中断する
\$ErrorActionPreference = "Stop"

# リモートの設定
\$RemoteHost = "psq-dev-ec2"
\$RemoteDir  = "/home/patesq/test-automation"

# Windowsのバックスラッシュをスラッシュに置換し、末尾の余分なスラッシュをトリム
\(CleanSpecPath =\)SpecPath.Replace('\', '/').TrimEnd('/')
\(FullLocalPath = "generated_testcode/\)CleanSpecPath"

# ローカル側での存在チェックとタイプの判別
\$PathType = "ファイル"
if (Test-Path \$FullLocalPath) {
    \(item = Get-Item\)FullLocalPath
    if (\(item -is [System.IO.DirectoryInfo]) {\)PathType = "ディレクトリ"
    }
} else {
    Write-Warning "⚠️ ローカル側にターゲットが見つかりません。リモート側のみに存在する可能性があります: \$FullLocalPath"
}

Write-Host "=========================================" -ForegroundColor Cyan
Write-Host "🖥️ EC2側でのリモートテスト実行を開始します..." -ForegroundColor Cyan
Write-Host "📂 ターゲットタイプ: \$PathType" -ForegroundColor Yellow
Write-Host "📄 ターゲットパス: \$FullLocalPath" -ForegroundColor Yellow
Write-Host "=========================================" -ForegroundColor Cyan

# --------------------------------------------------
# EC2側で実行するLinuxコマンドの組み立て
# --------------------------------------------------
# 前ステップの uv.lock があったため、デフォルトを `uv run` に設定しています。
# 別の環境の場合は、必要に応じて `pytest` や `source .venv/bin/activate && pytest` に変更してください。
\(RemoteCommand = "cd \){RemoteDir} && uv run pytest generated_testcode/\${CleanSpecPath}"

Write-Host "🏃 実行コマンド: ssh \${RemoteHost} `"${RemoteCommand}`"" -ForegroundColor Gray
Write-Host "-----------------------------------------" -ForegroundColor Gray

# SSH経由でコマンドを遠隔実行（リアルタイムにログが出力されます）
ssh \(RemoteHost "\)RemoteCommand"

Write-Host "=========================================" -ForegroundColor Green
Write-Host "✅ リモートテスト処理が終了しました。" -ForegroundColor Green
Write-Host "=========================================" -ForegroundColor Green

```

```
.\run-test-remote.ps1 -SpecPath "01.検索/2.プロフェッショナル検索"
```

### [重要] 実実装のための変更箇所
- RemoteCommandの箇所を，./scripts/run_tests.py ...に変更

## ダウンロード方法

リモートに入らず=local sshからコマンド接続するシェル

```
<#
.SYNOPSIS
    SSM/SSH経由でEC2上のpytestを遠隔実行し、生成されたレポートをローカルに自動ダウンロードします。
.EXAMPLE
    .\run-test.ps1 -SpecPath "auth/login_spec.xlsx"
#>
param (
    [Parameter(Mandatory=\$true, HelpMessage="テスト対象の仕様書パス、またはディレクトリを指定してください。")]
    [string]\$SpecPath
)

\$ErrorActionPreference = "Stop"

# リモート・ローカルの設定
\$RemoteHost = "psq-dev-ec2"
\$RemoteDir  = "/home/patesq/test-automation"
\$LocalReportDir = ".\test_reports" # ローカルのレポート保存先フォルダ

# レポートファイル名の定義（タイムスタンプ付き）
\$Timestamp  = Get-Date -Format "yyyyMMdd_HHmmss"
\(ReportName = "report_\)Timestamp"

# パスの補正
CleanSpecPath = SpecPath.Replace('\', '/').TrimEnd('/')

Write-Host "=========================================" -ForegroundColor Cyan
Write-Host "🖥️  EC2リモートテスト ＆ レポート取得を開始します..." -ForegroundColor Cyan
Write-Host "=========================================" -ForegroundColor Cyan

# --------------------------------------------------
# 1. EC2側での pytest 実行（HTML/XMLレポートを生成）
# --------------------------------------------------
# ※ `--html` と `--junitxml` オプションを追加し、リモートのルート直下に一時保存させます
RemoteCommand = "cd RemoteDir && uv run pytest generated_testcode/\$CleanSpecPath --html=ReportName.html --self-contained-html --junitxml=ReportName.xml"

Write-Host "🏃 テストを実行中..." -ForegroundColor Gray
# pytestがエラー（テスト失敗）で終了してもスクリプトを止めず、レポート取得を継続するために一時的にErrorActionを変更
\$OriginalEAP = \(ErrorActionPreference\)ErrorActionPreference = "Continue"

ssh RemoteHost "RemoteCommand"

ErrorActionPreference = OriginalEAP

# --------------------------------------------------
# 2. 生成されたレポートの自動ダウンロード
# --------------------------------------------------
Write-Host "-----------------------------------------" -ForegroundColor Gray
Write-Host "📥 レポートをローカルにダウンロード中..." -ForegroundColor Gray

# ローカルに保存先フォルダがなければ作成
if (-not (Test-Path \$LocalReportDir)) {
    New-Item -ItemType Directory -Path \$LocalReportDir | Out-Null
}

# scp を使って、EC2上のHTMLとXMLレポートをローカルの test_reports/ フォルダに取得
scp "RemoteHost:{RemoteDir}/ReportName.*" "LocalReportDir/"

# --------------------------------------------------
# 3. EC2側の一時レポートファイルを削除（クリーンアップ）
# --------------------------------------------------
ssh \$RemoteHost "rm -f RemoteDir/{ReportName}.*"

Write-Host "=========================================" -ForegroundColor Green
Write-Host "✨ 処理が完了しました！" -ForegroundColor Green
Write-Host "📊 レポート保存先: \$LocalReportDir\$ReportName.html" -ForegroundColor Yellow
Write-Host "=========================================" -ForegroundColor Green

```

- 前の実装にダウンロード処理だけ追加しただけか？gemini in chromeでのクイック生成のため別モデルで厳密評価予定

## [NEW] 上記3点処理ワンライナー

```
$spec="01.検索/2.プロフェッショナル検索" ; .\upload_code.ps1 -SpecPath $spec ; .\run-test-remote.ps1 -SpecPath $spec

```