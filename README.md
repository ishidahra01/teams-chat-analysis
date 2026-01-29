# teams-chat-analysis

**Delegated（ユーザー委任）で Graph の Teams 系スコープを付けて、実際にトークン取って Graph API 叩くまで**の手順を整理。
（Windows/Edge 前提・最短で動かす流れ＋Python例も付けます）

---

## 0) まず前提整理（Delegated の基本）

Delegated で Graph を叩く典型は **OAuth 2.0 Authorization Code + PKCE** です。

* ブラウザでユーザーがサインイン → アプリが **認可コード**を受け取る
* アプリがコードを **アクセストークン**に交換
* そのトークンで `https://graph.microsoft.com/...` を呼ぶ

---

## 1) Entra ID で「アプリ登録」

1. Azure portal → **Microsoft Entra ID**
2. **App registrations** → **New registration**
3. 入力

   * Name: 任意（例 `teams-graph-delegated-poc`）
   * Supported account types:

     * 同一テナントだけで良いなら **Single tenant**
   * Redirect URI:

     * ひとまず PoC なら **Public client / mobile & desktop** を選んで
       `http://localhost`（後で msal が使う）でもOK
       もしくは Web アプリなら `http://localhost:5000/getAToken`
4. **Register**

作成後、以下をメモ：

* **Application (client) ID**
* **Directory (tenant) ID**

---

## 2) Delegated の API Permissions を追加

アプリの画面で：

1. **API permissions** → **Add a permission**
2. **Microsoft Graph** → **Delegated permissions**
3. 必要なものを検索して追加（例）

### チャット & 会議

* `Chat.Read`
* `OnlineMeetings.Read`
* `Calendars.Read`（会議予定から辿るなら）

### Team / Channel

* `Team.ReadBasic.All`
* `ChannelMessage.Read.All` ← これがよく「管理者同意が必要」になる

### トランスクリプト（必要なら）

* `OnlineMeetingTranscript.Read.All`

4. 追加できたら **Grant admin consent**（管理者が必要なスコープがある場合）

   * `ChannelMessage.Read.All` などは **Admin Consent 必須**のことが多いです
   * 管理者権限がない場合は、管理者に「このアプリに admin consent して」と依頼が必要

> ここまでで「権限がアプリに付与された状態」になります。
> ただし **Delegated は “ユーザーが同意して初めて” 有効**なので、次でユーザー同意を発生させます。

---

## 3) 認証方式を決める（PoC 最短：Public client）

PoC で一番ラクなのは **Public client**（デスクトップ/CLI想定）です。

1. **Authentication**
2. **Platform configurations** を確認

   * “Mobile and desktop applications” が無ければ **Add a platform** → **Mobile and desktop**
3. **Allow public client flows** を **Yes**（表示される場合）
4. Redirect URI は以下どれかでOK

   * `http://localhost`
   * `msal{client_id}://auth`（使う方式により）

---

## 4) （任意だが推奨）Publisher verification / 検証周り

社内テナントだけなら必須ではないですが、運用するなら：

* **Branding** の設定
* **Verified publisher**（将来的に）

PoC はスキップでOK。

---

## 5) 実際にトークンを取って Graph を叩く（Python / MSAL）

### 5-1) インストール

```bash
pip install msal requests
```

### 5-2) そのまま動くサンプル（Device Code Flow）

ブラウザでユーザーにコード入力させる方式で、PoC が一番早いです。

```python
import msal
import requests

TENANT_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
CLIENT_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Scopes must match the delegated permissions you added in Entra ID
SCOPES = [
    "Chat.Read",
    "Team.ReadBasic.All",
    "ChannelMessage.Read.All",
    "OnlineMeetings.Read",
    "Calendars.Read",
    # "OnlineMeetingTranscript.Read.All",  # if needed
]

AUTHORITY = f"https://login.microsoftonline.com/{TENANT_ID}"

app = msal.PublicClientApplication(
    client_id=CLIENT_ID,
    authority=AUTHORITY,
)

flow = app.initiate_device_flow(scopes=SCOPES)
if "user_code" not in flow:
    raise RuntimeError(f"Failed to create device flow: {flow}")

print(flow["message"])  # User instructions

result = app.acquire_token_by_device_flow(flow)

if "access_token" not in result:
    raise RuntimeError(f"Failed to acquire token: {result}")

access_token = result["access_token"]

headers = {"Authorization": f"Bearer {access_token}"}

# Example 1: List chats the user is in
r = requests.get("https://graph.microsoft.com/v1.0/me/chats", headers=headers)
print("me/chats:", r.status_code)
print(r.text)

# Example 2: Joined teams
r = requests.get("https://graph.microsoft.com/v1.0/me/joinedTeams", headers=headers)
print("me/joinedTeams:", r.status_code)
print(r.text)
```

#### よくある詰まりポイント

* `ChannelMessage.Read.All` が入ってるのに 403 → **Admin consent がされていない**ことが多い
* `OnlineMeetings.Read` で `/me/onlineMeetings` が空 → ユーザーに会議がない or 取得範囲の問題
* トランスクリプトが取れない → Teams 側でトランスクリプト無効／会議側設定／権限不足

---

## 6) 「ユーザー同意」を確実に発生させたい場合（Auth URL）

Device code を使わず Web 認可（Authorization Code）で同意させたいなら、概念はこれです：

* authorize エンドポイントに `scope=...` を含めて飛ばす
* 初回に同意画面が出る
* 以後はトークン更新だけ

---

## 7) ここまで完了したら叩ける代表 API

* グループ/1:1 チャット

  * `GET /me/chats`
  * `GET /chats/{chat-id}/messages`
* Team / Channel

  * `GET /me/joinedTeams`
  * `GET /teams/{team-id}/channels`
  * `GET /teams/{team-id}/channels/{channel-id}/messages`
* 会議

  * `GET /me/events`（予定から）
  * `GET /me/onlineMeetings`
* トランスクリプト（取れる環境なら）

  * `GET /me/onlineMeetings/{id}/transcripts`

---
