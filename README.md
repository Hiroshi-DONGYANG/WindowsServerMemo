# Windows Server OpenSSH(SFTP)設定手順書

最終更新: 2026-11-10

Windows Server に OpenSSH Server を入れ、**ポート2222・公開鍵認証・SFTP専用ユーザー** で外部からファイル転送できるようにする手順。

## 目次

- [概要](#概要)
- [1. OpenSSHのインストール状態の確認](#1-opensshのインストール状態の確認)
- [2. OpenSSHのインストール](#2-opensshのインストール)
- [3. サービスの自動起動設定](#3-サービスの自動起動設定)
- [4. ポートの変更](#4-ポートの変更)
- [5. 秘密鍵の作成](#5-秘密鍵の作成)
- [6. sftpのアクセステスト](#6-sftpのアクセステスト)
- [トラブルシューティング](#トラブルシューティング)

---

## 概要

| 項目 | 設定値 |
| --- | --- |
| 待ち受けポート | `2222` |
| 認証方式 | 公開鍵のみ(パスワード認証は無効) |
| SFTPユーザー | `<<ユーザ名>>`(シェル不可・SFTP専用) |
| SFTPのルート | `<<ルートフォルダ>>`(例: `D:\sftp`) |
| 設定ファイル | `C:\ProgramData\ssh\sshd_config` |

> [!NOTE]
> - サーバー側のコマンドは、すべて **管理者権限のPowerShell** で実行する。
> - `<<ユーザ名>>` `<<ルートフォルダ>>` `<<サーバー名>>` は環境に合わせて置き換える。
> - Windows Server 2025 は OpenSSH が標準で組み込み済みのため、手順2は不要。

---

## 1. OpenSSHのインストール状態の確認

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
```

`OpenSSH.Server~~~~0.0.1.0` の `State` を確認する。

- `Installed` → 手順3へ
- `NotPresent` → 手順2でインストール

---

## 2. OpenSSHのインストール

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

> [!TIP]
> - インターネットに接続できないサーバーでは失敗する。FOD(Features on Demand)のISOを `-Source` で指定するか、GitHub の PowerShell/Win32-OpenSSH の MSI を使う。
> - Windows Server 2016 以前は標準機能に含まれないため、MSI版を使う。

---

## 3. サービスの自動起動設定

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

確認(`Status` が `Running`、`StartType` が `Automatic` ならOK):

```powershell
Get-Service sshd | Select-Object Name, Status, StartType
```

---

## 4. ポートの変更

### 4-1. sshd_config を編集

```powershell
notepad C:\ProgramData\ssh\sshd_config
```

`#Port 22` の先頭の `#` を消して書き換える。

```text
Port 2222
```

### 4-2. SFTP専用ユーザーとルートフォルダの設定

同じファイルの **いちばん最後** に追加する。

```text
Match User <<ユーザ名>>
    ChrootDirectory "C:\Program Files\FileMaker\FileMaker Server"
    ForceCommand internal-sftp
    AllowTcpForwarding no
    PermitTunnel no
```

- `ForceCommand internal-sftp` により SFTP 専用になる(ssh でのシェルログインは不可)。
- `ChrootDirectory` は SFTP 接続のときだけ有効。

> [!IMPORTANT]
> `Match` ブロックより後ろに書いた設定は、すべてそのブロックの中として扱われる。
> 全体に効かせたい設定は `Match` より前に書くこと。

### 4-3. ファイアウォールでポートを開ける

```powershell
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP-2222' -DisplayName 'OpenSSH Server (sshd) 2222' `
    -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 2222
```

> [!NOTE]
> AWS などのクラウド上にある場合は、セキュリティグループ側でも TCP 2222 を開ける。

### 4-4. 再起動と確認

```powershell
Restart-Service sshd
Get-NetTCPConnection -LocalPort 2222 -State Listen
```

新しいポートで接続できることを確認してから、旧ポート(22)の規則を無効にする。

```powershell
Disable-NetFirewallRule -Name 'OpenSSH-Server-In-TCP'
```

---

## 5. 秘密鍵の作成

サーバーで鍵ペアを作成 → 公開鍵を登録 → 秘密鍵を利用者に配布、の流れ。

### 5-1. 作業フォルダを用意

```powershell
$work = "C:\keygen"
New-Item -ItemType Directory -Path $work -Force
icacls $work /inheritance:r /grant "Administrators:F" /grant "SYSTEM:F"
```

### 5-2. 鍵ペアを作成

```powershell
ssh-keygen -t ed25519 -f "$work\<<ユーザ名>>" -C "<<ユーザ名>>@server 2026-11"
```

- `<<ユーザ名>>`(秘密鍵)と `<<ユーザ名>>.pub`(公開鍵)が作成される。
- `-C` は識別用のメモ。認証には使われない。
- パスフレーズを設定した場合は、鍵とは別の手段で伝える。
- 端末ごとに別の鍵を発行すると、紛失時にその端末の鍵だけ無効にできる。

> [!WARNING]
> 同じ名前で作り直すと上書きされる。作り直した場合は 5-3 の登録もやり直すこと。

### 5-3. 公開鍵を登録

```powershell
$user = "<<ユーザ名>>"
$dir  = "C:\Users\$user\.ssh"
New-Item -ItemType Directory -Path $dir -Force
Get-Content "$work\$user.pub" | Set-Content -Path "$dir\authorized_keys" -Encoding ascii
icacls "$dir\authorized_keys" /inheritance:r /grant "${user}:F" /grant "SYSTEM:F" /grant "Administrators:F"
```

- 鍵を追加する場合は `Set-Content` を `Add-Content` に変える(1行に1鍵)。
- Administrators グループのユーザーは `C:\ProgramData\ssh\administrators_authorized_keys` に登録する。
- プロファイルフォルダが `C:\Users\<<ユーザ名>>` 以外になっていないかも確認する。

登録した鍵の指紋を控えておく(6-2 で照合する)。

```powershell
ssh-keygen -lf "$dir\authorized_keys"
```

### 5-4. 秘密鍵を配布し、サーバーから削除

- 配布するのは拡張子なしの `<<ユーザ名>>` ファイル。
- メールへのそのままの添付は避ける(パスワード付きZIP+別経路でパスワードを伝える等)。
- 相手に届いたら、サーバー上の秘密鍵を削除する。

```powershell
Remove-Item "$work\<<ユーザ名>>"
```

---

## 6. sftpのアクセステスト

### 6-1. クライアント側で秘密鍵を配置

**Mac**

```bash
chmod 600 ~/.ssh/<<ユーザ名>>
```

**Windows**

```powershell
icacls "$env:USERPROFILE\.ssh\<<ユーザ名>>" /inheritance:r /grant "${env:USERNAME}:R"
```

### 6-2. 鍵が一致しているか確認

秘密鍵の指紋を表示し、サーバーで控えた指紋(5-3)と同じか確かめる。

```bash
ssh-keygen -lf ~/.ssh/<<ユーザ名>>
```

### 6-3. 接続テスト

```bash
sftp -P 2222 -i ~/.ssh/<<ユーザ名>> <<ユーザ名>>@<<サーバー名>>
sftp> pwd
```

- パスワードを聞かれずに入れ、`pwd` が `/` なら成功。
- sftp のポート指定は大文字の `-P`(ssh は小文字の `-p`)。
- `ssh` で接続すると `This service allows sftp connections only.` と出て切断されるが、SFTP専用設定による正常な動作。

### 6-4. パスワード認証を無効にする

> [!CAUTION]
> 鍵でログインできることを確認してから行うこと。

sshd_config の `Match` ブロックより前にある次の行を書き換える。

```text
PubkeyAuthentication yes
PasswordAuthentication no
```

```powershell
Restart-Service sshd
```

今の接続は切らずに、別のウィンドウから鍵でのログインをもう一度確認する。

---

## トラブルシューティング

| 症状 | 原因と対処 |
| --- | --- |
| 鍵を指定してもパスワードを聞かれる | `sftp -v ...` で `Offering public key` の後も認証が続くなら、サーバーが鍵を拒否している。まず指紋の一致(5-3・6-2)を確認 |
| 指紋が一致しない | 登録した鍵と配布した鍵が別物。クライアントで `ssh-keygen -y -f 秘密鍵` を実行して公開鍵を取り出し、`authorized_keys` に登録し直す |
| 指紋は一致するのに拒否される | `authorized_keys` のアクセス権(icacls)、置き場所(管理者は `administrators_authorized_keys`)、プロファイルフォルダ名を確認 |
| `UNPROTECTED PRIVATE KEY FILE` | クライアントの秘密鍵のアクセス権が緩い。6-1 を実行 |
| `This service allows sftp connections only.` | SFTP専用ユーザーに `ssh` で接続している。`sftp` コマンドを使う |
| 設定を変えても効かない | `Match` より後ろに書いていないか、行頭に `#` が残っていないか、`Restart-Service sshd` したかを確認 |

サーバー側のログ確認:

```powershell
Get-WinEvent -LogName OpenSSH/Operational -MaxEvents 50 | Format-List TimeCreated, Message
```

- 理由が出ないときは、sshd_config の `Match` より前に `LogLevel DEBUG3` を一時的に追加する(確認後は元に戻す)。
- サーバーの時刻は UTC のため、ログの時刻は日本時間の 9 時間前になる。
