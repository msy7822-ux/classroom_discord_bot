```txt
npm install
npm run dev
```

```txt
npm run deploy
```

[For generating/synchronizing types based on your Worker configuration run](https://developers.cloudflare.com/workers/wrangler/commands/#types):

```txt
npm run cf-typegen
```

## Discord Bot API の使用方法

### セットアップ

1. Discord Bot Tokenを取得
   - [Discord Developer Portal](https://discord.com/developers/applications)でアプリケーションを作成
   - Botを作成し、Tokenを取得
   - Botに必要な権限を付与（メッセージ送信権限など）

2. 環境変数の設定（ローカル開発）
   - `.dev.vars.example`を`.dev.vars`にコピー
   - `.dev.vars`ファイルに実際のDiscord Bot Tokenを設定

   ```bash
   cp .dev.vars.example .dev.vars
   # .dev.vars を編集して実際のTokenを設定
   ```

   - `.dev.vars`は`.gitignore`に含まれているため、Gitにコミットされません

3. 本番環境へのデプロイ時
   - `wrangler secret put DISCORD_BOT_TOKEN`コマンドで設定

4. Cron定期実行の設定
   - `wrangler.jsonc`の`triggers.crons`で実行スケジュールを設定
   - デフォルト: 毎週火曜日と金曜日のJST 17:00（UTC 8:00）
   - Cron形式: `分 時 日 月 曜日`（UTC時間）
   - 注意: Cloudflare WorkersのCronはUTCで実行されるため、JSTに合わせて調整が必要
   - 例: `0 8 * * 2` = 毎週火曜日UTC 8:00（火曜日JST 17:00）、`0 8 * * 5` = 毎週金曜日UTC 8:00（金曜日JST 17:00）

### API エンドポイント

#### POST /send

Discordチャンネルにシンプルなテキストメッセージを送信します。

**リクエスト例:**

```bash
curl -X POST https://your-worker.workers.dev/send \
  -H "Content-Type: application/json" \
  -d '{
    "channelId": "1234567890123456789",
    "message": "Hello from Discord Bot!"
  }'
```

**リクエストボディ:**

- `channelId` (string, 必須): DiscordチャンネルID
- `message` (string, 必須): 送信するメッセージ内容

#### POST /send/rich

Discordチャンネルにリッチメッセージ（埋め込みとボタンコンポーネント）を送信します。

**リクエスト例:**

```bash
curl -X POST http://localhost:8787/send/rich \
  -H "Content-Type: application/json" \
  -d '{
    "channelId": "1234567890123456789",
    "content": "📢 **進捗確認を記述してください。**",
    "embeds": [
      {
        "title": "進捗確認",
        "description": "以下のボタンから進捗状況を確認できます。",
        "color": 16753920,
        "fields": [
          {
            "name": "チェック状況",
            "value": "まだ確認されていません"
          }
        ],
        "timestamp": "2025-11-19T09:00:00.000Z"
      }
    ],
    "components": [
      {
        "type": 1,
        "components": [
          {
            "type": 2,
            "style": 1,
            "label": "確認済みにする",
            "custom_id": "checked_today"
          },
          {
            "type": 2,
            "style": 5,
            "label": "進捗を確認する",
            "url": "https://bk-realty.co.jp/latest"
          }
        ]
      }
    ]
  }'
```

**リクエストボディ:**

- `channelId` (string, 必須): DiscordチャンネルID
- `content` (string, オプション): 送信するテキストメッセージ内容
- `embeds` (array, オプション): 埋め込みメッセージの配列
  - `title` (string): 埋め込みのタイトル
  - `description` (string): 埋め込みの説明
  - `color` (number): 埋め込みの色（10進数、例: 16753920 = オレンジ）
  - `fields` (array): フィールドの配列
    - `name` (string): フィールド名
    - `value` (string): フィールドの値
    - `inline` (boolean): インライン表示するか
  - `timestamp` (string): ISO 8601形式のタイムスタンプ
- `components` (array, オプション): インタラクティブコンポーネント（ボタンなど）の配列
  - `type` (number): コンポーネントタイプ（1 = Action Row）
  - `components` (array): ボタンなどのコンポーネント配列
    - `type` (number): 2 = ボタン
    - `style` (number): ボタンスタイル（1 = Primary, 5 = Link）
    - `label` (string): ボタンのラベル
    - `custom_id` (string): カスタムID（インタラクション用、style=1の場合）
    - `url` (string): リンクURL（style=5の場合）

**注意**: このエンドポイントはリクエストボディなしで呼び出せます。テンプレートメッセージが自動的に送信されます。

**レスポンス例:**

成功時:

```json
{
  "success": true,
  "message": "Rich message sent successfully",
  "data": {
    "id": "1234567890123456789",
    "content": "📢 **進捗確認を記述してください。**",
    ...
  }
}
```

#### POST /interactions

Discordのインタラクション（ボタンクリックなど）を処理します。

**セットアップ:**

1. Discord Developer PortalでインタラクションURLを設定
   - [Discord Developer Portal](https://discord.com/developers/applications)でアプリケーションを開く
   - General Information → Interactions Endpoint URL に設定
   - ローカル: `http://localhost:8787/interactions`
   - 本番: デプロイ後のURL + `/interactions`

2. Botに必要な権限を付与
   - `applications.commands`スコープ
   - `bot`スコープ
   - 「メッセージを送信」「メッセージの管理」権限

**動作:**

- `checked_today`ボタンがクリックされると:
  - メッセージが更新され、「確認済み」状態になる
  - ボタンが無効化される
  - クリックしたユーザーに「✅ 進捗確認が完了しました！」という通知が表示される

#### Cron定期実行

Cronトリガーにより、定期的に進捗確認メッセージを自動送信します。

**設定:**

`wrangler.jsonc`の`triggers.crons`で実行スケジュールを設定:

```jsonc
{
  "triggers": {
    "crons": ["0 8 * * 2", "0 8 * * 5"], // 毎週火曜日と金曜日のJST 17:00（UTC 8:00）
  },
}
```

**Cron形式:**

- `分 時 日 月 曜日`（UTC時間）
- 曜日: 0=日曜日, 1=月曜日, 2=火曜日, 3=水曜日, 4=木曜日, 5=金曜日, 6=土曜日
- JST（UTC+9）で実行する場合のUTC時間の計算:
  - JST 17:00 = UTC 8:00（同じ日の8時）
  - 例:
    - `0 8 * * 2` - 毎週火曜日UTC 8:00 = 火曜日JST 17:00
    - `0 8 * * 5` - 毎週金曜日UTC 8:00 = 金曜日JST 17:00
    - `0 0 * * 1` - 毎週月曜日UTC 0時 = 火曜日JST 9時

**動作:**

- 設定された時刻に自動的に進捗確認メッセージが送信されます
- ログはCloudflare Dashboardで確認できます

**レスポンス例:**

成功時:

```json
{
  "success": true,
  "message": "Message sent successfully",
  "data": {
    "id": "1234567890123456789",
    "content": "Hello from Discord Bot!",
    ...
  }
}
```

エラー時:

```json
{
  "error": "Failed to send message to Discord",
  "details": {...}
}
```

### ローカル開発

1. `.dev.vars`ファイルを作成してTokenを設定

   ```bash
   cp .dev.vars.example .dev.vars
   # .dev.vars を編集して実際のTokenを設定
   ```

2. 開発サーバーを起動

   ```bash
   npm run dev
   ```

3. APIをテスト
   ```bash
   curl -X POST http://localhost:8787/send \
     -H "Content-Type: application/json" \
     -d '{
       "channelId": "YOUR_CHANNEL_ID",
       "message": "Hello from Discord Bot!"
     }'
   ```

### エラー対処

#### 401 Unauthorized

- `.dev.vars`ファイルの`DISCORD_BOT_TOKEN`が正しく設定されているか確認
- Discord Developer PortalでTokenが正しいか確認
- Tokenが再生成されていないか確認

#### 403 Forbidden（Botにチャンネルへのアクセス権限がない）

以下の手順で確認してください：

1. **Botがサーバーに招待されているか確認**
   - Discord Developer PortalでBotのOAuth2 URLを生成
   - サーバーにBotを招待（`applications.commands`、`bot`スコープが必要）

2. **Botの権限を確認**
   - サーバー設定 → ロール → Botのロールを確認
   - 以下の権限が必要：
     - ✅ メッセージを送信
     - ✅ チャンネルを見る
     - ✅ メッセージ履歴を読む

3. **チャンネルの権限を確認**
   - チャンネル設定 → 権限 → Botのロールを確認
   - Botがチャンネルにアクセスできるか確認
   - テキストチャンネルであることを確認（ボイスチャンネルやカテゴリチャンネルでは送信不可）

4. **チャンネルIDが正しいか確認**
   - 開発者モードを有効化
   - チャンネルを右クリック → 「IDをコピー」
   - APIリクエストの`channelId`と一致しているか確認

#### 404 Not Found

- チャンネルIDが正しいか確認
- 開発者モードを有効化してチャンネルIDをコピー
- チャンネルが存在し、アクセス可能か確認

# classroom_discord_bot
