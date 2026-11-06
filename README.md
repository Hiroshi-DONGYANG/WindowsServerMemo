# WindowsServerMemo


1. OpenSSH サーバーのインストール

Windows Server に標準搭載されているオンデマンド機能（Capabilities）を利用してインストールします。
powershell
# OpenSSH サーバーがインストール可能か確認
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'

# OpenSSH サーバーのインストール
Add-WindowsCapability -Online -Name "OpenSSH.Server~~~~0.0.1.0"
コードは注意してご使用ください。

2. サービス自動起動の設定と開始

OSの起動時に自動でSSHサービスが立ち上がるように設定し、サービスを開始します。
powershell
# サービス（sshd）の起動タイプを「自動」に変更
Set-Service -Name sshd -StartupType 'Automatic'

# サービスを開始
Start-Service -Name sshd

# 起動状態（Runningになっているか）の確認
Get-Service -Name sshd
コードは注意してご使用ください。

3. ファイアウォール規則の有効化

インストール時に自動で規則が追加されますが、SSH用のポート（デフォルト：22/tcp）が正常に許可されているか確認・有効化します。
powershell
# ファイアウォール規則の確認（「OpenSSH-Server-In-TCP」が True であること）
Get-NetFirewallRule -Name *ssh* | Select-Object Name, Enabled

# もし無効（False）だった場合は有効化を実行
Enable-NetFirewallRule -Name "OpenSSH-Server-In-TCP"
コードは注意してご使用ください。

4. 接続確認（テスト）

外部のPC（クライアント）のターミナル（PowerShellやコマンドプロンプト）から、以下のコマンドを実行して接続できるかテストします。
bash
# パスワード認証での接続テスト
ssh ユーザー名@サーバーのIPアドレス
コードは注意してご使用ください。
• ※ 初回接続時には、フィンガープリント（ECDSA key fingerprint）を信頼するか聞かれるため、yes と入力してください。
• ※ サーバー側のローカルアカウント、またはドメインアカウントのパスワードを入力するとログインできます。

💡 オプション：セキュリティ強化（公開鍵認証への移行）

パスワード認証を禁止し、公開鍵認証のみにする場合の追加設定です。
1. クライアント側で事前に作成した公開鍵（id_rsa.pub など）の内容をコピーします。
2. サーバー側のログイン対象ユーザーに応じて、以下のファイルをメモ帳等で開き、公開鍵テキストを追記します。
	• 一般ユーザーの場合: C:\Users\ユーザー名\.ssh\authorized_keys
	• 管理者（Administratorsグループ）の場合: C:\ProgramData\ssh\administrators_authorized_keys
3. サーバーの C:\ProgramData\ssh\sshd_config をテキストエディタで開き、以下の記述を修正・追加します。text
PubkeyAuthentication yes       # 有効化
PasswordAuthentication no      # パスワード認証を禁止
コードは注意してご使用ください。
4. 設定変更後、PowerShellで Restart-Service sshd を実行してSSHサービスを再起動します。
