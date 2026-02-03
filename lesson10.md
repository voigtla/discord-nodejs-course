# 第10回: Discord bot実コード完全読解

## この回で学ぶこと

- Discordイベントの流れ（messageCreateの裏側）
- すべての要素を統合した完全版Bot
- 各ファイルの役割と連携
- コードの精読（1行ずつ理解する）
- 拡張ポイント（次にやるべきこと）

## ゴール

この回を終えたあなたは、**「このBot、どう動いてる?」に答えられる**ようになります。

---

## 1. Discord Bot全体像の復習

### これまで学んだ要素

```
第1回: JavaScript / Node.js / イベント駆動
第2回: 変数・型・JSON
第3回: 関数と処理の分解
第4回: 条件分岐と繰り返し
第5回: 非同期処理とawait
第6回: SQLite基礎
第7回: トランザクションと安全なDB操作
第8回: Gemini APIとHTTPリクエスト
第9回: セキュリティとエラーハンドリング
```

### 最終的なBot構成

```
my-discord-bot/
├── .env                  (環境変数)
├── .gitignore            (Git除外設定)
├── package.json          (プロジェクト設定)
├── index.js              (起動ファイル)
├── config.js             (設定管理)
├── database.js           (DB操作)
├── gemini-api.js         (Gemini API)
├── security.js           (セキュリティ)
├── handlers/
│   └── messageHandler.js (メッセージ処理)
└── services/
    └── botService.js     (Bot処理)
```

---

## 2. データの流れ（完全版）

```
1. ユーザーがDiscordでメッセージ送信
   ↓
2. Discordサーバー → あなたのBot（messageCreateイベント発火）
   ↓
3. index.js: イベントを受信
   ↓
4. messageHandler.js: メッセージを判定・振り分け
   ↓
5. security.js: バリデーションとクールダウンチェック
   ↓
6. database.js: 過去の履歴を取得
   ↓
7. gemini-api.js: AI返答を生成
   ↓
8. database.js: やり取りを保存
   ↓
9. messageHandler.js: Discordに返信送信
   ↓
10. Discord → ユーザーに返信が表示される
```

---

## 3. 完全版コード：ファイル構成

### プロジェクトフォルダの作成

```
my-discord-bot/
├── .env
├── .gitignore
├── package.json
├── index.js
├── config.js
├── database.js
├── gemini-api.js
├── security.js
├── handlers/
│   └── messageHandler.js
└── services/
    └── botService.js
```

---

## 4. 完全版コード：各ファイルの実装

### .env（環境変数）

**ファイル**: `my-discord-bot/.env`

```env
# Discord Bot設定
DISCORD_TOKEN=あなたのDiscordBotトークン

# Gemini API設定
GEMINI_API_KEY=あなたのGeminiAPIキー

# Bot設定
BOT_PREFIX=!
MAX_HISTORY=10
COOLDOWN_SECONDS=3
```

### .gitignore（Git除外設定）

**ファイル**: `my-discord-bot/.gitignore`

```
# 環境変数
.env

# データベース
*.db
*.sqlite
*.sqlite3

# ログファイル
*.log

# 依存関係
node_modules/

# OS生成ファイル
.DS_Store
Thumbs.db
```

### package.json（プロジェクト設定）

**ファイル**: `my-discord-bot/package.json`

```json
{
  "name": "my-discord-bot",
  "version": "1.0.0",
  "description": "Discord Bot with Gemini AI and SQLite",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "discord.js": "^14.14.1",
    "dotenv": "^16.4.1",
    "better-sqlite3": "^9.3.0"
  },
  "author": "あなたの名前",
  "license": "MIT"
}
```

### config.js（設定管理）

**ファイル**: `my-discord-bot/config.js`

```javascript
require('dotenv').config();

// 設定値をまとめる
const config = {
  // Discord設定
  discord: {
    token: process.env.DISCORD_TOKEN
  },
  
  // Gemini API設定
  gemini: {
    apiKey: process.env.GEMINI_API_KEY,
    model: 'gemini-1.5-flash'
  },
  
  // Bot設定
  bot: {
    prefix: process.env.BOT_PREFIX || '!',
    maxHistory: parseInt(process.env.MAX_HISTORY) || 10,
    cooldownSeconds: parseInt(process.env.COOLDOWN_SECONDS) || 3
  }
};

// 必須設定のチェック
if (!config.discord.token) {
  console.error('❌ DISCORD_TOKENが設定されていません');
  process.exit(1);
}

if (!config.gemini.apiKey) {
  console.error('❌ GEMINI_API_KEYが設定されていません');
  process.exit(1);
}

module.exports = config;
```

### database.js（DB操作）

**ファイル**: `my-discord-bot/database.js`

```javascript
const Database = require('better-sqlite3');

let db = null;

// データベース接続
function getDatabase() {
  if (!db) {
    db = new Database('bot-data.db');
    db.pragma('journal_mode = WAL');
    console.log('✅ データベース接続確立');
  }
  return db;
}

// テーブル作成
function initDatabase() {
  const database = getDatabase();
  
  database.exec(`
    CREATE TABLE IF NOT EXISTS messages (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id TEXT NOT NULL,
      username TEXT NOT NULL,
      role TEXT NOT NULL,
      content TEXT NOT NULL,
      created_at TEXT NOT NULL
    )
  `);
  
  database.exec(`
    CREATE INDEX IF NOT EXISTS idx_messages_user_id ON messages(user_id)
  `);
  
  database.exec(`
    CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at)
  `);
  
  console.log('✅ データベース初期化完了');
}

// メッセージを保存（トランザクション）
function saveMessage(userId, username, role, content) {
  const database = getDatabase();
  
  const transaction = database.transaction(() => {
    const stmt = database.prepare(`
      INSERT INTO messages (user_id, username, role, content, created_at)
      VALUES (?, ?, ?, ?, ?)
    `);
    
    const now = new Date().toISOString();
    const info = stmt.run(userId, username, role, content, now);
    
    return info.lastInsertRowid;
  });
  
  try {
    const messageId = transaction();
    return { success: true, messageId };
  } catch (error) {
    console.error('❌ メッセージ保存エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// 会話履歴を取得
function getConversationHistory(userId, limit) {
  const database = getDatabase();
  
  const stmt = database.prepare(`
    SELECT role, content FROM messages
    WHERE user_id = ?
    ORDER BY created_at DESC
    LIMIT ?
  `);
  
  const messages = stmt.all(userId, limit);
  return messages.reverse();  // 古い順に並べ替え
}

// 会話履歴をクリア
function clearConversationHistory(userId) {
  const database = getDatabase();
  
  const transaction = database.transaction(() => {
    const stmt = database.prepare('DELETE FROM messages WHERE user_id = ?');
    const info = stmt.run(userId);
    return info.changes;
  });
  
  try {
    const deletedCount = transaction();
    return { success: true, deletedCount };
  } catch (error) {
    console.error('❌ 履歴削除エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// データベースを閉じる
function closeDatabase() {
  if (db) {
    db.close();
    db = null;
    console.log('✅ データベース接続を閉じました');
  }
}

// プロセス終了時のクリーンアップ
process.on('SIGINT', () => {
  closeDatabase();
  process.exit(0);
});

process.on('SIGTERM', () => {
  closeDatabase();
  process.exit(0);
});

module.exports = {
  initDatabase,
  saveMessage,
  getConversationHistory,
  clearConversationHistory,
  closeDatabase
};
```

### gemini-api.js（Gemini API）

**ファイル**: `my-discord-bot/gemini-api.js`

```javascript
const config = require('./config.js');

const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/${config.gemini.model}:generateContent?key=${config.gemini.apiKey}`;

// Gemini APIを呼び出す
async function callGeminiAPI(conversationHistory) {
  const contents = conversationHistory.map(msg => ({
    role: msg.role,
    parts: [{ text: msg.content }]
  }));
  
  const requestBody = {
    contents: contents,
    generationConfig: {
      temperature: 0.7,
      maxOutputTokens: 1000
    }
  };
  
  try {
    const response = await fetch(GEMINI_API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestBody)
    });
    
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      const errorMessage = errorData.error?.message || `HTTP ${response.status}`;
      throw new Error(errorMessage);
    }
    
    const data = await response.json();
    
    if (!data.candidates || data.candidates.length === 0) {
      throw new Error('APIからの返答がありません');
    }
    
    const aiText = data.candidates[0].content.parts[0].text;
    
    return { success: true, text: aiText };
    
  } catch (error) {
    console.error('❌ Gemini API エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// リトライ機能付きAPI呼び出し
async function callGeminiAPIWithRetry(conversationHistory, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const result = await callGeminiAPI(conversationHistory);
    
    if (result.success) {
      return result;
    }
    
    if (result.error.includes('429') && i < maxRetries - 1) {
      const waitTime = Math.pow(2, i) * 1000;
      console.log(`⏳ レート制限。${waitTime / 1000}秒待機...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    } else {
      return result;
    }
  }
  
  return { success: false, error: 'リトライ上限到達' };
}

module.exports = {
  callGeminiAPI,
  callGeminiAPIWithRetry
};
```

### security.js（セキュリティ）

**ファイル**: `my-discord-bot/security.js`

```javascript
const config = require('./config.js');

// クールダウン管理
const cooldowns = new Map();
const COOLDOWN_MS = config.bot.cooldownSeconds * 1000;

// クールダウンチェック
function checkCooldown(userId) {
  const now = Date.now();
  
  if (cooldowns.has(userId)) {
    const lastTime = cooldowns.get(userId);
    const timePassed = now - lastTime;
    
    if (timePassed < COOLDOWN_MS) {
      const remaining = Math.ceil((COOLDOWN_MS - timePassed) / 1000);
      return { allowed: false, remaining };
    }
  }
  
  cooldowns.set(userId, now);
  return { allowed: true };
}

// メッセージバリデーション
function validateMessage(content) {
  const errors = [];
  
  if (!content || content.trim().length === 0) {
    errors.push('メッセージが空です');
  }
  
  if (content.length > 2000) {
    errors.push('メッセージが長すぎます');
  }
  
  return { valid: errors.length === 0, errors };
}

// ログ出力
function logEvent(level, userId, username, message) {
  const timestamp = new Date().toISOString();
  const logMessage = `[${timestamp}] [${level}] ${username}(${userId}): ${message}`;
  
  if (level === 'ERROR') {
    console.error(logMessage);
  } else if (level === 'WARN') {
    console.warn(logMessage);
  } else {
    console.log(logMessage);
  }
}

module.exports = {
  checkCooldown,
  validateMessage,
  logEvent
};
```

### botService.js（Bot処理）

**ファイル**: `my-discord-bot/services/botService.js`

```javascript
const config = require('../config.js');
const database = require('../database.js');
const gemini = require('../gemini-api.js');
const security = require('../security.js');

// メッセージ処理
async function processMessage(message) {
  const userId = message.author.id;
  const username = message.author.username;
  const content = message.content;
  
  try {
    // 1. バリデーション
    const validation = security.validateMessage(content);
    if (!validation.valid) {
      await message.reply(`❌ ${validation.errors.join(', ')}`);
      return;
    }
    
    // 2. クールダウンチェック
    const cooldown = security.checkCooldown(userId);
    if (!cooldown.allowed) {
      await message.reply(`⏰ クールダウン中です。あと${cooldown.remaining}秒お待ちください。`);
      return;
    }
    
    // 3. ユーザーメッセージを保存
    database.saveMessage(userId, username, 'user', content);
    
    // 4. 会話履歴を取得
    const history = database.getConversationHistory(userId, config.bot.maxHistory);
    
    // 5. Gemini APIで返答生成
    security.logEvent('INFO', userId, username, `メッセージ送信: ${content.substring(0, 50)}`);
    
    const result = await gemini.callGeminiAPIWithRetry(history);
    
    if (!result.success) {
      await message.reply('❌ AI返答の生成に失敗しました。');
      security.logEvent('ERROR', userId, username, `API失敗: ${result.error}`);
      return;
    }
    
    // 6. Bot返答を保存
    database.saveMessage(userId, 'Bot', 'model', result.text);
    
    // 7. Discordに返信
    await message.reply(result.text);
    
    security.logEvent('INFO', userId, username, `返答送信完了`);
    
  } catch (error) {
    console.error('❌ メッセージ処理エラー:', error);
    await message.reply('エラーが発生しました。').catch(() => {});
  }
}

// ヘルプコマンド
function getHelpMessage() {
  return `
📖 **Bot使い方**
普通にメッセージを送ると、AIが返答します。

**コマンド**:
\`${config.bot.prefix}help\` - このヘルプを表示
\`${config.bot.prefix}clear\` - 会話履歴をクリア
\`${config.bot.prefix}ping\` - Botの応答確認
  `.trim();
}

// クリアコマンド
async function clearHistory(message) {
  const userId = message.author.id;
  const result = database.clearConversationHistory(userId);
  
  if (result.success) {
    await message.reply(`🗑️ 会話履歴を${result.deletedCount}件削除しました。`);
  } else {
    await message.reply('❌ 履歴削除に失敗しました。');
  }
}

module.exports = {
  processMessage,
  getHelpMessage,
  clearHistory
};
```

### messageHandler.js（メッセージハンドラ）

**ファイル**: `my-discord-bot/handlers/messageHandler.js`

```javascript
const config = require('../config.js');
const botService = require('../services/botService.js');

// メッセージ受信時の処理
async function handleMessage(message) {
  // Bot自身のメッセージは無視
  if (message.author.bot) {
    return;
  }
  
  const content = message.content.trim();
  
  // コマンド判定
  if (content.startsWith(config.bot.prefix)) {
    await handleCommand(message, content);
  } else {
    // 通常メッセージとして処理
    await botService.processMessage(message);
  }
}

// コマンド処理
async function handleCommand(message, content) {
  const command = content.toLowerCase();
  
  if (command === `${config.bot.prefix}help`) {
    const helpMessage = botService.getHelpMessage();
    await message.reply(helpMessage);
  } else if (command === `${config.bot.prefix}clear`) {
    await botService.clearHistory(message);
  } else if (command === `${config.bot.prefix}ping`) {
    const start = Date.now();
    const reply = await message.reply('🏓 Pong!');
    const latency = Date.now() - start;
    await reply.edit(`🏓 Pong! (レイテンシ: ${latency}ms)`);
  } else {
    await message.reply(`❌ 不明なコマンドです。\`${config.bot.prefix}help\` でヘルプを確認してください。`);
  }
}

module.exports = {
  handleMessage
};
```

### index.js（起動ファイル）

**ファイル**: `my-discord-bot/index.js`

```javascript
const { Client, GatewayIntentBits } = require('discord.js');
const config = require('./config.js');
const database = require('./database.js');
const messageHandler = require('./handlers/messageHandler.js');

// Discord Clientの作成
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent
  ]
});

// データベース初期化
database.initDatabase();

// Bot起動時のイベント
client.once('ready', () => {
  console.log('✅ Botが起動しました');
  console.log(`📛 ユーザー名: ${client.user.tag}`);
  console.log(`🆔 Bot ID: ${client.user.id}`);
  console.log(`🔢 サーバー数: ${client.guilds.cache.size}`);
  console.log('\n🤖 Botが待機中...\n');
});

// メッセージ受信時のイベント
client.on('messageCreate', async (message) => {
  try {
    await messageHandler.handleMessage(message);
  } catch (error) {
    console.error('❌ メッセージ処理エラー:', error);
  }
});

// エラーハンドリング
client.on('error', (error) => {
  console.error('❌ Discord クライアントエラー:', error);
});

process.on('unhandledRejection', (error) => {
  console.error('❌ 未処理のPromise拒否:', error);
});

// Bot起動
console.log('🚀 Botを起動しています...\n');
client.login(config.discord.token);
```

---

## 5. コードの精読：各部分の役割

### index.js: エントリーポイント

```javascript
// 1. Discord Clientを作成
const client = new Client({ intents: [...] });
// → Discord との接続を確立するクライアント

// 2. データベース初期化
database.initDatabase();
// → テーブルが存在しなければ作成

// 3. 起動完了イベント
client.once('ready', () => { ... });
// → Bot が Discord に接続したら実行される

// 4. メッセージ受信イベント
client.on('messageCreate', async (message) => { ... });
// → メッセージが送信されるたびに実行される

// 5. ログイン
client.login(config.discord.token);
// → Discord API に接続
```

### messageHandler.js: メッセージの振り分け

```javascript
// 1. Bot自身のメッセージを無視
if (message.author.bot) return;
// → 無限ループを防ぐ

// 2. コマンドか通常メッセージかを判定
if (content.startsWith(config.bot.prefix)) {
  handleCommand(message, content);
} else {
  botService.processMessage(message);
}
// → 処理を適切な関数に振り分け
```

### botService.js: 実際の処理

```javascript
// 1. バリデーション
const validation = security.validateMessage(content);
// → 入力値が正しいかチェック

// 2. クールダウンチェック
const cooldown = security.checkCooldown(userId);
// → 連続送信を制限

// 3. ユーザーメッセージを保存
database.saveMessage(userId, username, 'user', content);
// → DB に保存（トランザクション）

// 4. 会話履歴を取得
const history = database.getConversationHistory(userId, limit);
// → 過去のやり取りを取得

// 5. AI返答を生成
const result = await gemini.callGeminiAPIWithRetry(history);
// → Gemini API にリクエスト（リトライ機能付き）

// 6. Bot返答を保存
database.saveMessage(userId, 'Bot', 'model', result.text);
// → AI の返答も DB に保存

// 7. Discordに返信
await message.reply(result.text);
// → ユーザーに返信を送信
```

---

## 6. 実行方法

### ステップ1: 依存関係をインストール

```bash
cd my-discord-bot
npm install
```

### ステップ2: .envファイルを設定

```env
DISCORD_TOKEN=あなたのDiscordBotトークン
GEMINI_API_KEY=あなたのGeminiAPIキー
BOT_PREFIX=!
MAX_HISTORY=10
COOLDOWN_SECONDS=3
```

### ステップ3: Botを起動

```bash
npm start
```

### 実行結果

```
🚀 Botを起動しています...

✅ データベース接続確立
✅ データベース初期化完了
✅ Botが起動しました
📛 ユーザー名: MyBot#1234
🆔 Bot ID: 123456789012345678
🔢 サーバー数: 2

🤖 Botが待機中...

[2024-01-01T10:00:00.000Z] [INFO] 太郎(123456): メッセージ送信: こんにちは
[2024-01-01T10:00:02.500Z] [INFO] 太郎(123456): 返答送信完了
```

---

## 7. Bot動作の完全フロー

### シナリオ: ユーザー「太郎」が「こんにちは」と送信

```
1. Discord: 太郎が「こんにちは」と送信
   ↓
2. index.js: messageCreateイベント発火
   client.on('messageCreate', async (message) => { ... })
   ↓
3. messageHandler.js: メッセージを受信
   handleMessage(message)
   ↓
4. messageHandler.js: コマンドではないと判定
   botService.processMessage(message)
   ↓
5. botService.js: バリデーション実行
   security.validateMessage('こんにちは')
   → { valid: true }
   ↓
6. botService.js: クールダウンチェック
   security.checkCooldown('123456')
   → { allowed: true }
   ↓
7. database.js: ユーザーメッセージを保存
   saveMessage('123456', '太郎', 'user', 'こんにちは')
   → トランザクション実行 → DB に INSERT
   ↓
8. database.js: 会話履歴を取得
   getConversationHistory('123456', 10)
   → [{ role: 'user', content: 'こんにちは' }]
   ↓
9. gemini-api.js: Gemini APIにリクエスト
   callGeminiAPIWithRetry(history)
   → HTTPリクエスト送信
   → レスポンス受信
   → { success: true, text: 'こんにちは！...' }
   ↓
10. database.js: Bot返答を保存
    saveMessage('123456', 'Bot', 'model', 'こんにちは！...')
    → トランザクション実行 → DB に INSERT
    ↓
11. botService.js: Discordに返信
    message.reply('こんにちは！...')
    → Discord API にリクエスト
    ↓
12. Discord: 太郎に返信が表示される
```

**処理時間**: 約2〜3秒

---

## 8. 拡張ポイント

### 拡張1: 画像認識

Gemini APIは画像も受け付けられます。

```javascript
// 画像付きメッセージの処理
if (message.attachments.size > 0) {
  const attachment = message.attachments.first();
  if (attachment.contentType.startsWith('image/')) {
    // 画像をBase64に変換してGemini APIに送信
  }
}
```

### 拡張2: スラッシュコマンド

Discord の新しいコマンドシステム。

```javascript
const { SlashCommandBuilder } = require('discord.js');

const command = new SlashCommandBuilder()
  .setName('ask')
  .setDescription('AIに質問する')
  .addStringOption(option =>
    option.setName('question')
      .setDescription('質問内容')
      .setRequired(true)
  );
```

### 拡張3: マルチサーバー対応

現在はユーザーIDのみで管理していますが、サーバーごとに会話を分ける。

```javascript
// サーバーIDとユーザーIDを組み合わせる
const uniqueId = `${message.guild.id}-${message.author.id}`;
```

### 拡張4: データベースのバックアップ

定期的にDBをバックアップ。

```javascript
const fs = require('fs');
const schedule = require('node-schedule');

// 毎日午前3時にバックアップ
schedule.scheduleJob('0 3 * * *', () => {
  const backupPath = `backup-${Date.now()}.db`;
  fs.copyFileSync('bot-data.db', backupPath);
  console.log('✅ バックアップ完了:', backupPath);
});
```

### 拡張5: Web管理画面

Expressで管理画面を作成。

```javascript
const express = require('express');
const app = express();

app.get('/stats', (req, res) => {
  // 統計情報を表示
  const stats = database.getStats();
  res.json(stats);
});

app.listen(3000, () => {
  console.log('管理画面: http://localhost:3000');
});
```

---

## 9. まとめ：この講座で学んだこと

### 第1回〜第10回の総復習

| 回 | テーマ | 重要ポイント |
|----|--------|------------|
| 1 | JavaScript / Node.js | イベント駆動の概念 |
| 2 | 変数・型・JSON | データの構造化 |
| 3 | 関数と処理の分解 | handler と service |
| 4 | 条件分岐と繰り返し | filter と map |
| 5 | 非同期処理と await | Promise の理解 |
| 6 | SQLite基礎 | データベースの必要性 |
| 7 | トランザクションと安全性 | データ整合性 |
| 8 | Gemini API | HTTPリクエストとレスポンス |
| 9 | セキュリティ | バリデーションとエラーハンドリング |
| 10 | 完全版Bot | すべての統合 |

### あなたができるようになったこと

✅ Discord Botのコードを怖がらずに読める  
✅ SQLiteを「なぜ使っているか」論理的に説明できる  
✅ Gemini APIの構造を理解し、自分の言葉で扱える  
✅ セキュリティ上の「地雷」を察知し、自らブレーキをかけられる  
✅ 「このBot、どう動いてる?」に答えられる

---

## 10. 次のステップ

### 推奨学習パス

1. **TypeScript への移行**: より型安全なコードを書く
2. **テストの追加**: Jest でユニットテストを書く
3. **CI/CD の構築**: GitHub Actions で自動デプロイ
4. **Dockerize**: コンテナ化してデプロイを簡単に
5. **モニタリング**: Prometheusなどで監視

### 参考リソース

- **Discord.js 公式ドキュメント**: https://discord.js.org/
- **Gemini API ドキュメント**: https://ai.google.dev/docs
- **better-sqlite3 ドキュメント**: https://github.com/WiseLibs/better-sqlite3
- **Node.js ベストプラクティス**: https://github.com/goldbergyoni/nodebestpractices

---

## 11. おわりに

### この講座の目的

この講座は、**「技術の裏側にある理屈」を理解する**ことを目的としていました。

コピペでは、問題が起きたときに対処できません。  
しかし、理屈が分かれば、**自分で考えて解決できる**ようになります。

### 最後のメッセージ

あなたは今、**技術を理解する人**になりました。

この知識を使って、自分だけのBotを作り、  
さらに学び続けてください。

プログラミングは、**終わりのない旅**です。  
楽しんでください。

---

**完全版ソースコードは、各ファイルのセクションを参照してください。**

すべてのファイルを正しく配置し、.envを設定すれば、  
Discord Botが完全に動作します。

**これで、全10回の講座は完結です。お疲れ様でした！**

---

## 付録: トラブルシューティング

### Bot が起動しない

```
❌ DISCORD_TOKENが設定されていません
```

**対処法**: .envファイルにトークンを設定

### Bot がメッセージに反応しない

**対処法**: Discord Developer Portalで「Message Content Intent」を有効化

### API呼び出しエラー

```
❌ Gemini API エラー: 401
```

**対処法**: Gemini APIキーを確認

### データベースエラー

```
❌ メッセージ保存エラー: database is locked
```

**対処法**: WALモードが有効か確認（database.js参照）

---

**この講座を最後まで読んでくださり、ありがとうございました！**
