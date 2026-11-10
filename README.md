Windows Server OpenSSH(SFTP)設定手順書
2026年11月10日 · @Hiroshi Higashionna
概要
Windows Server に OpenSSH Server を入れ、ポート2222・公開鍵認証・SFTP専用ユーザーで外部からファイル転送できるようにする手順です。
項目
設定値
待ち受けポート
2222
認証方式
公開鍵のみ(パスワード認証は無効)
SFTPユーザー
<<ユーザ名>>(シェル不可・SFTP専用)
SFTPのルート
<<ルートフォルダ>>(例: D:\sftp)
設定ファイル
C:\ProgramData\ssh\sshd_config
• サーバー側のコマンドは、すべて管理者権限のPowerShellで実行します。
• <<ユーザ名>> <<ルートフォルダ>> <<サーバー名>> は環境に合わせて置き換えます。
• Windows Server 2025 は OpenSSH が標準で組み込み済みのため、手順2は不要です。
1. OpenSSHのインストール状態の確認
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
OpenSSH.Server~~~~0.0.1.0 の State を確認します。
• Installed → 手順3へ進む
• NotPresent → 手順2でインストールする
2. OpenSSHのインストール
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
• インターネットに接続できないサーバーでは失敗します。その場合は FOD(Features on Demand)のISOを -Source で指定するか、GitHubの PowerShell/Win32-OpenSSH のMSIを使います。
• Windows Server 2016以前は標準機能に含まれないため、MSI版を使います。
3. サービスの自動起動設定
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
確認します。Status が Running、StartType が Automatic ならOKです。
Get-Service sshd | Select-Object Name, Status, StartType
4. ポートの変更
4-1. sshd_config を編集
notepad C:\ProgramData\ssh\sshd_config
#Port 22 を次のように書き換えます(先頭の # を消す)。
Port 2222
4-2. SFTP専用ユーザーとルートフォルダの設定
同じファイルのいちばん最後に追加します。
Match User <<ユーザ名>>
    ChrootDirectory <<ルートフォルダ>>
    ForceCommand internal-sftp
    AllowTcpForwarding no
    PermitTunnel no
• ForceCommand internal-sftp により、このユーザーはSFTP専用になります(sshでのシェルログインは不可)。
• ChrootDirectory はSFTP接続のときだけ効きます。
• Match ブロックより後ろに書いた設定は、すべてそのブロックの中として扱われます。共通の設定は Match より前に書きます。
4-3. ファイアウォールでポートを開ける
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP-2222' -DisplayName 'OpenSSH Server (sshd) 2222' `
    -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 2222
AWSなどのクラウド上にある場合は、セキュリティグループなどでも TCP 2222 を開けます。
4-4. 再起動と確認
Restart-Service sshd
Get-NetTCPConnection -LocalPort 2222 -State Listen
新しいポートで接続できることを確認してから、旧ポート(22)の規則を無効にします。
Disable-NetFirewallRule -Name 'OpenSSH-Server-In-TCP'
5. 秘密鍵の作成
サーバーで鍵ペアを作り、公開鍵を登録したうえで、秘密鍵を利用者に配布します。
5-1. 作業フォルダを用意
$work = "C:\keygen"
New-Item -ItemType Directory -Path $work -Force
icacls $work /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
5-2. 鍵ペアを作成
ssh-keygen -t ed25519 -f "$work\<<ユーザ名>>" -C "<<ユーザ名>>@server 2026-11"
• <<ユーザ名>>(秘密鍵)と <<ユーザ名>>.pub(公開鍵)ができます。
• -C は識別用のメモです。認証には使われません。
• パスフレーズを設定した場合は、鍵とは別の手段で伝えます。
• 端末ごとに別の鍵を発行すると、紛失時にその端末の鍵だけを無効にできます。
• **同じ名前で作り直すと上書きされます。**作り直した場合は、5-3の登録もやり直してください。
5-3. 公開鍵を登録
$user = "<<ユーザ名>>"
$dir  = "C:\Users\$user\.ssh"
New-Item -ItemType Directory -Path $dir -Force
Get-Content "$work\$user.pub" | Set-Content -Path "$dir\authorized_keys" -Encoding ascii
icacls "$dir\authorized_keys" /inheritance:r /grant "${user}:F" /grant "SYSTEM:F" /grant "Administrators:F"
• 鍵を追加する場合は Set-Content を Add-Content に変えます(1行に1鍵)。
• Administratorsグループのユーザーは、C:\ProgramData\ssh\administrators_authorized_keys に登録します。
• プロファイルフォルダが C:\Users\<<ユーザ名>> 以外になっていないかも確認します。
登録した鍵の指紋を確認しておきます(6-2で照合します)。
ssh-keygen -lf "$dir\authorized_keys"
5-4. 秘密鍵を配布し、サーバーから削除
• 配布するのは拡張子なしの <<ユーザ名>> ファイルです。
• メールへのそのままの添付は避け、パスワード付きZIP＋別経路でパスワードを伝えるなどします。
• 相手に届いたら、サーバー上の秘密鍵を削除します。
Remove-Item "$work\<<ユーザ名>>"
6. sftpのアクセステスト
6-1. クライアント側で秘密鍵を配置
Mac
chmod 600 ~/.ssh/<<ユーザ名>>
Windows
icacls "$env:USERPROFILE\.ssh\<<ユーザ名>>" /inheritance:r /grant "${env:USERNAME}:R"
6-2. 鍵が一致しているか確認
秘密鍵の指紋を表示し、サーバーで確認した指紋(5-3)と同じか確かめます。
ssh-keygen -lf ~/.ssh/<<ユーザ名>>
6-3. 接続テスト
sftp -P 2222 -i ~/.ssh/<<ユーザ名>> <<ユーザ名>>@<<サーバー名>>
sftp> pwd
• パスワードを聞かれずに入れ、pwd が / なら成功です。
• sftpのポート指定は大文字の -P です(sshは小文字の -p)。
• ssh で接続すると「This service allows sftp connections only.」と出て切断されますが、SFTP専用設定による正常な動作です。
6-4. パスワード認証を無効にする
鍵でログインできたことを確認してから行います。sshd_config の Match ブロックより前にある次の行を書き換えます。
PubkeyAuthentication yes
PasswordAuthentication no
Restart-Service sshd
今の接続は切らずに、別のウィンドウから鍵でのログインをもう一度確認します。
トラブルシューティング
症状
原因と対処
鍵を指定してもパスワードを聞かれる
sftp -v ... で Offering public key の後に認証が続くなら、サーバーが鍵を拒否しています。まず指紋の一致(5-3・6-2)を確認します
指紋が一致しない
鍵の作り直しなどで、登録した鍵と配布した鍵が別物です。クライアントで ssh-keygen -y -f 秘密鍵 を実行して公開鍵を取り出し、authorized_keys に登録し直します
指紋は一致するのに拒否される
authorized_keys のアクセス権(icacls)、置き場所(管理者は administrators_authorized_keys)、プロファイルフォルダ名を確認します
UNPROTECTED PRIVATE KEY FILE
クライアントの秘密鍵のアクセス権が緩いです。6-1を実行します
This service allows sftp connections only.
SFTP専用ユーザーに ssh で接続しています。sftp コマンドを使います
設定を変えても効かない
Match ブロックより後ろに書いていないか、行頭に # が残っていないか、Restart-Service sshd をしたか確認します
サーバー側のログは次のコマンドで確認できます。理由が出ないときは、sshd_config の Match より前に LogLevel DEBUG3 を一時的に追加します。
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 50 | Format-List TimeCreated, Message
サーバーの時刻はUTCのため、ログの時刻は日本時間の9時間前になります。
