# 第6回: SQLite基礎 (なぜDBを使うのか)

## この回で学ぶこと

- SQLiteとは何か（軽量データベースの正体）
- ファイル保存との違い（なぜJSONファイルではダメなのか）
- テーブル・行・列の概念（データの構造化）
- SELECT/INSERT/UPDATE/DELETEの基本
- なぜSQLiteが選ばれているのか

## ゴール

この回を終えたあなたは、**「なぜSQLiteが選ばれているか」を説明できる**ようになります。

---

## 1. データベースとは何か？

### データベースの本質：「構造化されたデータの保管庫」

データベース（DB）は、**データを効率的に保存・検索・更新するための仕組み**です。

### なぜデータベースが必要なのか？

Discord Botには、以下のようなデータを保存する必要があります：

```
- ユーザーの会話履歴
- Bot設定値
- コマンド実行回数
- ユーザーごとの設定
```

これらを**プログラムのメモリに保存すると、再起動で消える**ため、  
永続的に保存する仕組みが必要です。

---

## 2. ファイル保存との違い

### 方法1: JSONファイルに保存

```javascript
// messages.json
[
  { "user": "太郎", "message": "こんにちは", "time": "2024-01-01 10:00" },
  { "user": "花子", "message": "おはよう", "time": "2024-01-01 10:01" }
]
```

### JSONファイルの問題点

#### 問題1: 検索が遅い

```javascript
// 「太郎」のメッセージだけを取得したい
const fs = require('fs');
const data = JSON.parse(fs.readFileSync('messages.json', 'utf8'));

// 全データを読み込んで、filterで絞り込む
const taroMessages = data.filter(msg => msg.user === "太郎");
```

データが10万件あると、**毎回10万件すべてを読み込む必要がある**

#### 問題2: 同時書き込みで壊れる

```javascript
// ユーザーAとBが同時にメッセージを送った場合
ユーザーA: ファイルを読む → データ追加 → ファイルに書く
ユーザーB: ファイルを読む → データ追加 → ファイルに書く
            ↑
         Aの書き込みが反映される前に読んでしまう
         → Aのデータが消える
```

#### 問題3: データの整合性が保証されない

```javascript
// 書き込み中にエラーが起きると...
{
  "user": "太郎",
  "message": "こんに  ← ここで停電
  // ファイルが壊れる
```

### 方法2: データベース（SQLite）に保存

```sql
-- テーブル: messages
| id | user | message      | time             |
|----|------|--------------|------------------|
| 1  | 太郎 | こんにちは   | 2024-01-01 10:00 |
| 2  | 花子 | おはよう     | 2024-01-01 10:01 |
```

### データベースの利点

✅ **高速検索**: インデックスで特定のデータをすぐ見つけられる  
✅ **同時アクセス対応**: ロック機構で同時書き込みを制御  
✅ **データ整合性**: トランザクションでエラー時の復旧が可能  
✅ **SQL言語**: 統一された言語で操作できる

---

## 3. SQLiteとは何か？

### SQLiteの特徴

SQLiteは、**ファイル1つで完結する軽量データベース**です。

```
通常のデータベース（MySQL, PostgreSQLなど）:
- サーバープロセスが必要
- ネットワーク経由でアクセス
- 大規模・複数クライアント向け

SQLite:
- ファイル1つ（data.db）
- プロセス内で直接アクセス
- 小〜中規模・単一アプリ向け
```

### SQLiteが選ばれる理由

| 項目 | SQLite | MySQL/PostgreSQL |
|------|--------|------------------|
| セットアップ | 不要 | サーバー構築が必要 |
| 管理コスト | 低い | 高い |
| パフォーマンス | 小規模なら高速 | 大規模向け |
| 適用範囲 | 個人Bot、モバイルアプリ | Webサービス、企業システム |

**Discord Botには、SQLiteが最適**

---

## 4. テーブル・行・列の概念

### データベースの構造

```
データベース (data.db)
  ├── テーブル1: users
  ├── テーブル2: messages
  └── テーブル3: settings
```

### テーブル = Excelのシート

```
テーブル: messages

| id  | user_id | content      | created_at       |
|-----|---------|--------------|------------------|
| 1   | 123     | こんにちは   | 2024-01-01 10:00 |
| 2   | 456     | おはよう     | 2024-01-01 10:01 |
| 3   | 123     | 元気？       | 2024-01-01 10:02 |

↑     ↑       ↑              ↑
行     列       列              列
(row)  (column) (column)        (column)
```

### 各要素の役割

- **テーブル**: 関連するデータをまとめたもの（例: messages, users）
- **行（row）**: 1件のデータ（例: 1つのメッセージ）
- **列（column）**: データの項目（例: user_id, content）
- **主キー（Primary Key）**: 各行を一意に識別するID（例: id列）

---

## 5. SQL言語の基礎

### SQLとは？

SQL（Structured Query Language）は、**データベースを操作するための言語**です。

### 基本的な4つの操作（CRUD）

| 操作 | 意味 | SQL文 |
|------|------|-------|
| Create | 作成 | INSERT |
| Read | 読取 | SELECT |
| Update | 更新 | UPDATE |
| Delete | 削除 | DELETE |

---

## 6. テーブルの作成（CREATE TABLE）

### テーブル定義

```sql
CREATE TABLE messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id TEXT NOT NULL,
  username TEXT NOT NULL,
  content TEXT NOT NULL,
  created_at TEXT NOT NULL
);
```

### 各部の説明

```sql
CREATE TABLE messages (           -- messagesという名前のテーブルを作成
  id INTEGER PRIMARY KEY AUTOINCREMENT,  -- id: 整数、主キー、自動増加
  user_id TEXT NOT NULL,          -- user_id: 文字列、必須
  username TEXT NOT NULL,         -- username: 文字列、必須
  content TEXT NOT NULL,          -- content: 文字列、必須
  created_at TEXT NOT NULL        -- created_at: 文字列、必須
);
```

### データ型

| SQLiteの型 | 説明 | 例 |
|-----------|------|-----|
| INTEGER | 整数 | 1, 100, -5 |
| REAL | 小数 | 3.14, -0.5 |
| TEXT | 文字列 | "こんにちは", "2024-01-01" |
| BLOB | バイナリデータ | 画像、ファイル |

---

## 7. データの挿入（INSERT）

### 基本構文

```sql
INSERT INTO messages (user_id, username, content, created_at)
VALUES ('123456', '太郎', 'こんにちは', '2024-01-01 10:00:00');
```

### 複数行を一度に挿入

```sql
INSERT INTO messages (user_id, username, content, created_at)
VALUES 
  ('123456', '太郎', 'こんにちは', '2024-01-01 10:00:00'),
  ('789012', '花子', 'おはよう', '2024-01-01 10:01:00'),
  ('123456', '太郎', '元気？', '2024-01-01 10:02:00');
```

### Node.jsでの実行例

```javascript
const Database = require('better-sqlite3');
const db = new Database('data.db');

// データを挿入
const stmt = db.prepare(`
  INSERT INTO messages (user_id, username, content, created_at)
  VALUES (?, ?, ?, ?)
`);

stmt.run('123456', '太郎', 'こんにちは', '2024-01-01 10:00:00');
```

**`?`はプレースホルダー**（後から値を埋め込む）

---

## 8. データの取得（SELECT）

### 全データを取得

```sql
SELECT * FROM messages;
```

実行結果:
```
| id | user_id | username | content      | created_at       |
|----|---------|----------|--------------|------------------|
| 1  | 123456  | 太郎     | こんにちは   | 2024-01-01 10:00 |
| 2  | 789012  | 花子     | おはよう     | 2024-01-01 10:01 |
| 3  | 123456  | 太郎     | 元気？       | 2024-01-01 10:02 |
```

### 特定の列だけ取得

```sql
SELECT username, content FROM messages;
```

実行結果:
```
| username | content      |
|----------|--------------|
| 太郎     | こんにちは   |
| 花子     | おはよう     |
| 太郎     | 元気？       |
```

### 条件付きで取得（WHERE）

```sql
SELECT * FROM messages WHERE user_id = '123456';
```

実行結果:
```
| id | user_id | username | content      | created_at       |
|----|---------|----------|--------------|------------------|
| 1  | 123456  | 太郎     | こんにちは   | 2024-01-01 10:00 |
| 3  | 123456  | 太郎     | 元気？       | 2024-01-01 10:02 |
```

### 並び替え（ORDER BY）

```sql
-- 新しい順
SELECT * FROM messages ORDER BY created_at DESC;

-- 古い順
SELECT * FROM messages ORDER BY created_at ASC;
```

### 件数制限（LIMIT）

```sql
-- 最新5件を取得
SELECT * FROM messages ORDER BY created_at DESC LIMIT 5;
```

---

## 9. データの更新（UPDATE）

### 基本構文

```sql
UPDATE messages
SET content = '修正されたメッセージ'
WHERE id = 1;
```

**⚠️ 注意**: WHEREを忘れると、**全行が更新される**

### 複数の列を更新

```sql
UPDATE messages
SET 
  content = '修正されたメッセージ',
  created_at = '2024-01-01 11:00:00'
WHERE id = 1;
```

---

## 10. データの削除（DELETE）

### 基本構文

```sql
DELETE FROM messages WHERE id = 1;
```

**⚠️ 注意**: WHEREを忘れると、**全行が削除される**

### 複数行を削除

```sql
-- 特定ユーザーのメッセージをすべて削除
DELETE FROM messages WHERE user_id = '123456';
```

### すべて削除

```sql
DELETE FROM messages;
```

---

## 11. 実践：SQLiteでメッセージ履歴を管理

### プロジェクトフォルダの作成

```
my-discord-bot/
├── database.js
├── test-db.js
└── data.db (自動生成)
```

### ステップ1: better-sqlite3のインストール

```bash
cd my-discord-bot
npm install better-sqlite3
```

### ステップ2: database.js（DB操作）

**ファイル**: `my-discord-bot/database.js`

```javascript
const Database = require('better-sqlite3');

// データベース接続
const db = new Database('data.db');

// テーブル作成
function initDatabase() {
  const createTable = `
    CREATE TABLE IF NOT EXISTS messages (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id TEXT NOT NULL,
      username TEXT NOT NULL,
      content TEXT NOT NULL,
      created_at TEXT NOT NULL
    )
  `;
  
  db.exec(createTable);
  console.log("✅ データベース初期化完了");
}

// メッセージを保存
function saveMessage(userId, username, content) {
  const stmt = db.prepare(`
    INSERT INTO messages (user_id, username, content, created_at)
    VALUES (?, ?, ?, ?)
  `);
  
  const now = new Date().toISOString();
  const info = stmt.run(userId, username, content, now);
  
  return {
    id: info.lastInsertRowid,
    userId,
    username,
    content,
    createdAt: now
  };
}

// 全メッセージを取得
function getAllMessages() {
  const stmt = db.prepare('SELECT * FROM messages ORDER BY created_at ASC');
  return stmt.all();
}

// 特定ユーザーのメッセージを取得
function getMessagesByUser(userId) {
  const stmt = db.prepare(`
    SELECT * FROM messages 
    WHERE user_id = ? 
    ORDER BY created_at ASC
  `);
  return stmt.all(userId);
}

// 最新N件を取得
function getRecentMessages(limit) {
  const stmt = db.prepare(`
    SELECT * FROM messages 
    ORDER BY created_at DESC 
    LIMIT ?
  `);
  return stmt.all(limit);
}

// メッセージを削除
function deleteMessage(id) {
  const stmt = db.prepare('DELETE FROM messages WHERE id = ?');
  const info = stmt.run(id);
  return info.changes > 0;
}

// 特定ユーザーのメッセージをすべて削除
function clearUserMessages(userId) {
  const stmt = db.prepare('DELETE FROM messages WHERE user_id = ?');
  const info = stmt.run(userId);
  return info.changes;
}

// 全メッセージを削除
function clearAllMessages() {
  const stmt = db.prepare('DELETE FROM messages');
  const info = stmt.run();
  return info.changes;
}

// データベースを閉じる
function closeDatabase() {
  db.close();
  console.log("✅ データベース接続を閉じました");
}

// 外部に公開
module.exports = {
  initDatabase,
  saveMessage,
  getAllMessages,
  getMessagesByUser,
  getRecentMessages,
  deleteMessage,
  clearUserMessages,
  clearAllMessages,
  closeDatabase
};
```

### ステップ3: test-db.js（テスト）

**ファイル**: `my-discord-bot/test-db.js`

```javascript
const db = require('./database.js');

console.log("=== SQLiteテスト開始 ===\n");

// 1. データベース初期化
console.log("1. データベース初期化");
db.initDatabase();
console.log("");

// 2. メッセージを保存
console.log("2. メッセージを保存");
const msg1 = db.saveMessage('123456', '太郎', 'こんにちは');
console.log("保存:", msg1);

const msg2 = db.saveMessage('789012', '花子', 'おはよう');
console.log("保存:", msg2);

const msg3 = db.saveMessage('123456', '太郎', '元気？');
console.log("保存:", msg3);

const msg4 = db.saveMessage('345678', '次郎', 'こんばんは');
console.log("保存:", msg4);
console.log("");

// 3. 全メッセージを取得
console.log("3. 全メッセージを取得");
const allMessages = db.getAllMessages();
console.log(`総数: ${allMessages.length}件`);
allMessages.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 4. 特定ユーザーのメッセージを取得
console.log("4. 太郎のメッセージを取得");
const taroMessages = db.getMessagesByUser('123456');
console.log(`太郎のメッセージ: ${taroMessages.length}件`);
taroMessages.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.content}`);
});
console.log("");

// 5. 最新2件を取得
console.log("5. 最新2件を取得");
const recentMessages = db.getRecentMessages(2);
recentMessages.reverse().forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 6. メッセージを削除
console.log("6. ID=1のメッセージを削除");
const deleted = db.deleteMessage(1);
console.log(`削除成功: ${deleted}`);
console.log("");

// 7. 削除後の全メッセージ
console.log("7. 削除後の全メッセージ");
const afterDelete = db.getAllMessages();
console.log(`総数: ${afterDelete.length}件`);
afterDelete.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 8. 特定ユーザーのメッセージをクリア
console.log("8. 太郎のメッセージをクリア");
const clearedCount = db.clearUserMessages('123456');
console.log(`削除件数: ${clearedCount}件`);
console.log("");

// 9. クリア後の全メッセージ
console.log("9. クリア後の全メッセージ");
const afterClear = db.getAllMessages();
console.log(`総数: ${afterClear.length}件`);
afterClear.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 10. データベースを閉じる
db.closeDatabase();

console.log("=== SQLiteテスト完了 ===");
```

### 実行方法

```bash
# プロジェクトフォルダに移動
cd my-discord-bot

# 実行
node test-db.js
```

### 実行結果

```
=== SQLiteテスト開始 ===

1. データベース初期化
✅ データベース初期化完了

2. メッセージを保存
保存: {
  id: 1,
  userId: '123456',
  username: '太郎',
  content: 'こんにちは',
  createdAt: '2024-01-01T01:00:00.000Z'
}
保存: { id: 2, userId: '789012', username: '花子', content: 'おはよう', ... }
保存: { id: 3, userId: '123456', username: '太郎', content: '元気？', ... }
保存: { id: 4, userId: '345678', username: '次郎', content: 'こんばんは', ... }

3. 全メッセージを取得
総数: 4件
  [1] 太郎: こんにちは
  [2] 花子: おはよう
  [3] 太郎: 元気？
  [4] 次郎: こんばんは

4. 太郎のメッセージを取得
太郎のメッセージ: 2件
  [1] こんにちは
  [3] 元気？

5. 最新2件を取得
  [3] 太郎: 元気？
  [4] 次郎: こんばんは

6. ID=1のメッセージを削除
削除成功: true

7. 削除後の全メッセージ
総数: 3件
  [2] 花子: おはよう
  [3] 太郎: 元気？
  [4] 次郎: こんばんは

8. 太郎のメッセージをクリア
削除件数: 1件

9. クリア後の全メッセージ
総数: 2件
  [2] 花子: おはよう
  [4] 次郎: こんばんは

✅ データベース接続を閉じました
=== SQLiteテスト完了 ===
```

---

## 12. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── database.js
├── test-db.js
└── data.db (自動生成)
```

### database.js（完成版）

**ファイルの場所**: `my-discord-bot/database.js`

```javascript
const Database = require('better-sqlite3');

// データベース接続
const db = new Database('data.db');

// テーブル作成
function initDatabase() {
  const createTable = `
    CREATE TABLE IF NOT EXISTS messages (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id TEXT NOT NULL,
      username TEXT NOT NULL,
      content TEXT NOT NULL,
      created_at TEXT NOT NULL
    )
  `;
  
  db.exec(createTable);
  console.log("✅ データベース初期化完了");
}

// メッセージを保存
function saveMessage(userId, username, content) {
  const stmt = db.prepare(`
    INSERT INTO messages (user_id, username, content, created_at)
    VALUES (?, ?, ?, ?)
  `);
  
  const now = new Date().toISOString();
  const info = stmt.run(userId, username, content, now);
  
  return {
    id: info.lastInsertRowid,
    userId,
    username,
    content,
    createdAt: now
  };
}

// 全メッセージを取得
function getAllMessages() {
  const stmt = db.prepare('SELECT * FROM messages ORDER BY created_at ASC');
  return stmt.all();
}

// 特定ユーザーのメッセージを取得
function getMessagesByUser(userId) {
  const stmt = db.prepare(`
    SELECT * FROM messages 
    WHERE user_id = ? 
    ORDER BY created_at ASC
  `);
  return stmt.all(userId);
}

// 最新N件を取得
function getRecentMessages(limit) {
  const stmt = db.prepare(`
    SELECT * FROM messages 
    ORDER BY created_at DESC 
    LIMIT ?
  `);
  return stmt.all(limit);
}

// メッセージを削除
function deleteMessage(id) {
  const stmt = db.prepare('DELETE FROM messages WHERE id = ?');
  const info = stmt.run(id);
  return info.changes > 0;
}

// 特定ユーザーのメッセージをすべて削除
function clearUserMessages(userId) {
  const stmt = db.prepare('DELETE FROM messages WHERE user_id = ?');
  const info = stmt.run(userId);
  return info.changes;
}

// 全メッセージを削除
function clearAllMessages() {
  const stmt = db.prepare('DELETE FROM messages');
  const info = stmt.run();
  return info.changes;
}

// メッセージ数をカウント
function countMessages() {
  const stmt = db.prepare('SELECT COUNT(*) as count FROM messages');
  const result = stmt.get();
  return result.count;
}

// ユーザー別のメッセージ数をカウント
function countMessagesByUser(userId) {
  const stmt = db.prepare('SELECT COUNT(*) as count FROM messages WHERE user_id = ?');
  const result = stmt.get(userId);
  return result.count;
}

// データベースを閉じる
function closeDatabase() {
  db.close();
  console.log("✅ データベース接続を閉じました");
}

// 外部に公開
module.exports = {
  initDatabase,
  saveMessage,
  getAllMessages,
  getMessagesByUser,
  getRecentMessages,
  deleteMessage,
  clearUserMessages,
  clearAllMessages,
  countMessages,
  countMessagesByUser,
  closeDatabase
};
```

### test-db.js（完成版）

**ファイルの場所**: `my-discord-bot/test-db.js`

```javascript
const db = require('./database.js');

console.log("=== SQLiteテスト開始 ===\n");

// 1. データベース初期化
console.log("【1】データベース初期化");
db.initDatabase();
console.log("");

// 2. メッセージを保存
console.log("【2】メッセージを保存");
const msg1 = db.saveMessage('123456', '太郎', 'こんにちは');
console.log(`✅ 保存完了 (ID: ${msg1.id})`);

const msg2 = db.saveMessage('789012', '花子', 'おはよう');
console.log(`✅ 保存完了 (ID: ${msg2.id})`);

const msg3 = db.saveMessage('123456', '太郎', '元気？');
console.log(`✅ 保存完了 (ID: ${msg3.id})`);

const msg4 = db.saveMessage('345678', '次郎', 'こんばんは');
console.log(`✅ 保存完了 (ID: ${msg4.id})`);
console.log("");

// 3. 全メッセージを取得
console.log("【3】全メッセージを取得");
const allMessages = db.getAllMessages();
console.log(`📊 総数: ${allMessages.length}件`);
allMessages.forEach(msg => {
  const time = new Date(msg.created_at).toLocaleTimeString();
  console.log(`  [${msg.id}] [${time}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 4. 特定ユーザーのメッセージを取得
console.log("【4】太郎のメッセージを取得");
const taroMessages = db.getMessagesByUser('123456');
console.log(`📊 太郎のメッセージ: ${taroMessages.length}件`);
taroMessages.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.content}`);
});
console.log("");

// 5. 最新2件を取得
console.log("【5】最新2件を取得");
const recentMessages = db.getRecentMessages(2);
console.log(`📊 最新2件:`);
recentMessages.reverse().forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 6. メッセージ数をカウント
console.log("【6】メッセージ数をカウント");
const totalCount = db.countMessages();
const taroCount = db.countMessagesByUser('123456');
console.log(`📊 総メッセージ数: ${totalCount}件`);
console.log(`📊 太郎のメッセージ数: ${taroCount}件`);
console.log("");

// 7. メッセージを削除
console.log("【7】ID=1のメッセージを削除");
const deleted = db.deleteMessage(1);
console.log(`✅ 削除${deleted ? '成功' : '失敗'}`);
console.log("");

// 8. 削除後の全メッセージ
console.log("【8】削除後の全メッセージ");
const afterDelete = db.getAllMessages();
console.log(`📊 総数: ${afterDelete.length}件`);
afterDelete.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 9. 特定ユーザーのメッセージをクリア
console.log("【9】太郎のメッセージをクリア");
const clearedCount = db.clearUserMessages('123456');
console.log(`✅ ${clearedCount}件を削除`);
console.log("");

// 10. クリア後の全メッセージ
console.log("【10】クリア後の全メッセージ");
const afterClear = db.getAllMessages();
console.log(`📊 総数: ${afterClear.length}件`);
afterClear.forEach(msg => {
  console.log(`  [${msg.id}] ${msg.username}: ${msg.content}`);
});
console.log("");

// 11. データベースを閉じる
db.closeDatabase();

console.log("=== SQLiteテスト完了 ===");
```

---

## 13. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] データベースは「構造化されたデータの保管庫」
- [ ] JSONファイルと違い、検索が速く、同時アクセスに強い
- [ ] SQLiteはファイル1つで完結する軽量DB
- [ ] テーブル、行、列の概念を理解した
- [ ] SELECT/INSERT/UPDATE/DELETEの基本を理解した
- [ ] プレースホルダー（?）でSQLインジェクションを防げる

### 次回予告

**第7回: Node.js × SQLite (実装と危険)**

次回は、「データベースの危険性」を学びます。

- Node.jsからのアクセス方法
- 同時アクセス問題
- トランザクションの重要性
- 「DBが壊れるコード」を見抜く方法

---

## 14. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: なぜJSONファイルではなくSQLiteを使うの？

<details>
<summary>答えの例</summary>

JSONファイルは、データが増えると検索が遅くなり、同時書き込みでデータが壊れる危険がある。  
SQLiteは、インデックスで高速検索ができ、ロック機構で同時アクセスを制御でき、トランザクションでデータ整合性を保証できる。  
小〜中規模のBotには、SQLiteが最適。

</details>

### Q2: プレースホルダー（?）はなぜ必要？

<details>
<summary>答えの例</summary>

直接文字列を埋め込むと、SQLインジェクション攻撃の危険がある。  
例: `DELETE FROM messages WHERE id = ${userInput}`  
ユーザーが「1 OR 1=1」と入力すると、全データが削除される。  
プレースホルダーを使えば、値が安全にエスケープされる。

</details>

### Q3: PRIMARY KEY AUTOINCREMENTは何をしているの？

<details>
<summary>答えの例</summary>

PRIMARY KEYは「各行を一意に識別するキー」。  
AUTOINCREMENTは「自動で連番を振る」機能。  
データを挿入するたびに、idが1, 2, 3...と自動で増える。  
これにより、各メッセージに固有のIDが付けられる。

</details>

---

これらの質問に答えられれば、第6回は完璧です！

次回、[第7回: Node.js × SQLite (実装と危険)](./lesson07.md) でお会いしましょう。

---

**学習メモ**: SQLiteのデータは、data.dbファイルに保存されます。このファイルを削除すると、すべてのデータが消えます。
