# teams-chat-analysis

Microsoft Graph (Delegated) を使って、Teams のチャネル / チャット / 会議まわりのデータをローカルに落として解析するためのサンプルです。

メインのエントリーポイントは `01_get_teams_messages.ipynb` で、次のような情報を取得します：

- 参加している Team と、その配下のチャネル一覧
- 特定チャネルの全メッセージ（＋必要に応じてスレッド返信）
- 1:1 / グループ / 会議チャットのメッセージ
- オンライン会議のメタデータ・参加者
- オンライン会議のトランスクリプト (VTT)

（Windows / Edge + Python ローカル実行を前提とした最短パスの PoC 用サンプルです）

---

## 1. 01_get_teams_messages.ipynb で何をしているか

### 1-1. 認証（Device Code Flow）

- `msal` の PublicClientApplication + Device Code Flow でユーザー委任トークンを取得します。
- 使用スコープ（Delegated）：
  - `User.Read`
  - `Calendars.Read`
  - `Chat.Read`
  - `OnlineMeetings.Read`
  - `OnlineMeetingTranscript.Read.All`
  - `Team.ReadBasic.All`
  - `Channel.ReadBasic.All`
  - `ChannelMessage.Read.All`
- トークン取得後、動作確認として次を呼び出します：
  - `GET /me/chats`
  - `GET /me/joinedTeams`

### 1-2. チーム / チャネル / チャットの一覧取得

- 参加している Team 一覧：`GET /me/joinedTeams`
- 各 Team のチャネル一覧：`GET /teams/{team-id}/channels`
- グループチャット一覧：`GET /me/chats?$filter=chatType eq 'group'`
- 会議チャット一覧：`GET /me/chats?$filter=chatType eq 'meeting'`
- ここで出力された `id` や `onlineMeetingInfo.joinWebUrl` を、後続セルの ID 設定に利用します。

### 1-3. ID 設定

一覧の出力を見ながら、次の値をノートブック内の「ID 設定セル」にコピペします：

- `team_id` / `channel_id`（チャネルメッセージ取得用）
- `group_chat_id`（グループチャットメッセージ取得用）
- `meeting_chat_id`（会議チャットメッセージ取得用）
- `join_url`（オンライン会議特定用 `joinWebUrl`）

### 1-4. メッセージ取得と JSON 保存

ノートブック内では、ページング対応ヘルパー `get_with_paging` と、JSON 保存ヘルパー `save_messages_to_json` を使って次を行います。

- チャネルメッセージ
  - `GET /teams/{team-id}/channels/{channel-id}/messages`
  - （必要なら）`GET /teams/{team-id}/channels/{channel-id}/messages/{message-id}/replies`
  - 結果は `output/channel_messages_YYYYMMDD_HHMMSS.json` に保存
- グループチャットメッセージ
  - `GET /chats/{chat-id}/messages`（`group_chat_id` を指定）
  - 結果は `output/group_chat_messages_YYYYMMDD_HHMMSS.json` に保存
- 会議チャットメッセージ
  - `GET /chats/{chat-id}/messages`（`meeting_chat_id` を指定）
  - 結果は `output/meeting_chat_messages_YYYYMMDD_HHMMSS.json` に保存

### 1-5. オンライン会議とトランスクリプト

- join URL から特定会議を取得：
  - `GET /me/onlineMeetings?$filter=JoinWebUrl eq '{joinWebUrl}'`
- 会議オブジェクトから参加者情報を表示
- トランスクリプト一覧：
  - `GET /me/onlineMeetings/{meeting-id}/transcripts`
- トランスクリプト本文 (VTT)：
  - `GET /me/onlineMeetings/{meeting-id}/transcripts/{transcript-id}/content`
  - 取得した内容を `meeting_transcript_0.vtt` として保存

※ トランスクリプト系 API はテナント設定や権限によっては 4xx (特に 403 / 400) になる場合があります。その場合はテナント管理者に Graph 権限・会議ポリシーを確認してください。

---

## 2. 前提・必要な設定

### 2-1. 環境

- Windows + Edge を想定
- Python 3.10 以降
- `pip` が利用できること

### 2-2. Entra ID でのアプリ登録

1. Azure portal → **Microsoft Entra ID**
2. **App registrations** → **New registration**
3. 次を入力：
   - Name: 任意（例 `teams-graph-delegated-poc`）
   - Supported account types: 自テナントだけで良いなら **Single tenant**
   - Redirect URI: オプション（`http://localhost` を登録とかでもOK）
4. **Register** を押下

作成後、以下を控えておきます：

- Application (client) ID
- Directory (tenant) ID

### 2-3. Delegated の API Permissions

アプリ登録画面で：

1. **API permissions** → **Add a permission**
2. **Microsoft Graph** → **Delegated permissions**
3. 次のスコープを追加（Notebook で使用）

| 用途                         | 代表的なエンドポイント                                          | 主な Delegated 権限                 |
|------------------------------|------------------------------------------------------------------|--------------------------------------|
| サインイン / ユーザー情報    | `GET /me`                                                       | `User.Read`                          |
| チャット一覧 / メッセージ    | `GET /me/chats`, `GET /chats/{chat-id}/messages`               | `Chat.Read`                          |
| カレンダー / 会議予定        | `GET /me/events` など                                         | `Calendars.Read`                    |
| 参加している Team 一覧       | `GET /me/joinedTeams`                                          | `Team.ReadBasic.All`                |
| Team 内のチャネル一覧       | `GET /teams/{team-id}/channels`                               | `Channel.ReadBasic.All`             |
| チャネルメッセージ / 返信    | `GET /teams/{team-id}/channels/{channel-id}/messages` など     | `ChannelMessage.Read.All`*          |
| オンライン会議の取得        | `GET /me/onlineMeetings`、`...?$filter=JoinWebUrl eq '...'`   | `OnlineMeetings.Read`               |
| 会議トランスクリプト一覧/本文| `GET /me/onlineMeetings/{id}/transcripts`、`.../content`      | `OnlineMeetingTranscript.Read.All`* |

`*` の付いているスコープは、多くのテナントで **管理者同意 (admin consent)** が必須です。

4. 必要なスコープを追加し終えたら **Grant admin consent** を実行
   - 自分に管理者権限がない場合は、テナント管理者に admin consent を依頼してください。

> Delegated 権限は「アプリにスコープが付いている」だけでは有効にならず、**ユーザーが実際に同意したときに初めて有効**になります。Notebook 実行時の Device Code Flow でユーザー同意が行われます。

### 2-4. 認証方式（Public client）の有効化

- 対象アプリの **Authentication** 画面で、**Allow public client flows** を **Enabled** にします。
- これにより Device Code Flow での委任トークン取得が可能になります。

---

## 3. ローカル環境の準備

### 3-1. ライブラリのインストール

```bash
pip install msal requests
```

### 3-2. Notebook の設定

1. このリポジトリをクローンして開きます。
2. `01_get_teams_messages.ipynb` を開き、最初の「クライアント設定」セルで次を自分の値に変更します。
   - `TENANT_ID`（Directory (tenant) ID）
   - `CLIENT_ID`（Application (client) ID）

---

## 4. Notebook の実行手順（README 形式の手順書）

1. **トークン取得 & 動作確認**
   - 「クライアント設定」セルを実行します。
   - コンソールに表示される Device Code Flow の案内に従い、ブラウザでサインイン＆同意を行います。
   - `me/chats` / `me/joinedTeams` のレスポンスが 200 になっていることを確認します。

2. **チーム / チャネル / チャットの一覧取得**
   - 「Teams チャネル/グループチャット/会議データの取得」セクションのセルを順番に実行します。
   - 出力された一覧から、対象としたい Team / チャネル / グループチャット / 会議チャットを決めます。

3. **ID の設定**
   - 「ID 設定セル」を開き、一覧の出力から次の値をコピペして設定します。
     - `team_id`
     - `channel_id`
     - `group_chat_id`
     - `meeting_chat_id`
     - `join_url`（onlineMeetingInfo.joinWebUrl）

4. **特定チャネルのメッセージ取得**
   - 「特定チャネルの全メッセージ取得」セルを実行します。
   - 標準出力にメッセージのサマリが表示され、同時に `output/channel_messages_*.json` が作成されます。

5. **特定グループチャットのメッセージ取得**
   - 「特定グループチャットの全メッセージ取得」セルを実行します。
   - 標準出力にメッセージのサマリが表示され、`output/group_chat_messages_*.json` が作成されます。

6. **オンライン会議の情報・トランスクリプト・会議チャット取得**
   - 「オンライン会議のメタデータ・参加者・トランスクリプト・会議チャット」セクションのセルを実行します。
   - `join_url` に対応する会議の参加者が表示されます。
   - トランスクリプト API を利用できる環境であれば、`meeting_transcript_0.vtt` が作成されます。
   - 会議チャットのメッセージは `output/meeting_chat_messages_*.json` として保存されます。

7. **取得データの活用**
   - `output/*.json` は別 Notebook で pandas に読み込んで集計・可視化に利用できます。
   - `meeting_transcript_0.vtt` は会議の字幕ログなので、テキスト抽出・要約・キーフレーズ抽出などの分析に利用できます。

---

## 5. 備考（Delegated の基本イメージ）

- Delegated では、ユーザーがサインインして同意した権限の範囲で Graph API を呼び出します。
- Web アプリでは典型的に **OAuth 2.0 Authorization Code + PKCE** を使いますが、このサンプルでは CLI / Notebook で手軽に試せる **Device Code Flow** を採用しています。
- 本サンプルで取得したトークンやレスポンスボディにはユーザー/組織の機微情報が含まれうるため、取り扱いには注意してください。
