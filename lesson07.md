# 第7回: Node.js × SQLite (実装と危険)

## この回で学ぶこと

- Node.jsからのSQLiteアクセス方法
- 同時アクセス問題とその対策
- トランザクションの重要性（全部成功 or 全部失敗）
- 排他制御とロック機構
- 「DBが壊れるコード」の見抜き方

## ⚠️ 重要度：高

この回では、**データベースを壊してしまう危険なコード**を学びます。  
実際のBot開発で最もトラブルが起きやすい部分なので、じっくり理解してください。

## ゴール

この回を終えたあなたは、**「DBが壊れるコード」を見抜ける**ようになります。

---

## 1. 同時アクセス問題とは？

### 問題の本質：「同時に書き込むと壊れる」

Discord Botは、**複数のユーザーから同時にメッセージを受け取ります**。

```
時刻 10:00:00.000  → ユーザーA: "こんにちは"
時刻 10:00:00.001  → ユーザーB: "おはよう"
時刻 10:00:00.002  → ユーザーC: "こんばんは"
```

これらが**ほぼ同時に**データベースに書き込まれようとします。

### 同時アクセスの危険性

```javascript
// ❌ 危険なコード（簡略版）
function saveMessage(message) {
  // 1. 現在のメッセージ数を取得
  const count = db.getCount();  // count = 10
  
  // 2. 新しいIDを計算
  const newId = count + 1;      // newId = 11
  
  // 3. メッセージを保存
  db.insert(newId, message);
}
```

### 何が起きるか？

```
【シナリオ】
時刻 0ms:
  ユーザーA → count = 10 を取得
  ユーザーB → count = 10 を取得 (Aの保存前)

時刻 10ms:
  ユーザーA → newId = 11 で保存
  ユーザーB → newId = 11 で保存 (同じID！)

結果:
  → IDが重複
  → データベースエラー or データ上書き
```

---

## 2. 解決策1: AUTOINCREMENT（自動採番）

### SQLiteの機能を使う

```sql
CREATE TABLE messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,  -- ← これが重要
  content TEXT
);
```

**AUTOINCREMENTを使えば、SQLiteが自動で一意のIDを振る**

### 仕組み

```
ユーザーA → INSERT (id指定なし) → SQLiteが自動で id=11 を割り当て
ユーザーB → INSERT (id指定なし) → SQLiteが自動で id=12 を割り当て
```

**IDの重複が起きない**

---

## 3. 解決策2: トランザクション

### トランザクションとは？

トランザクションは、**複数の操作をまとめて「全部成功 or 全部失敗」にする仕組み**です。

### 日常の例：銀行振込

```
【トランザクションなし】
1. Aの口座から1万円引き落とす  ← 成功
2. Bの口座に1万円入金する     ← エラー！

結果: Aの1万円が消えた（最悪）

【トランザクションあり】
BEGIN TRANSACTION;
  1. Aの口座から1万円引き落とす  ← 成功
  2. Bの口座に1万円入金する     ← エラー！
ROLLBACK;  ← 全部なかったことにする

結果: 何も変わっていない（安全）
```

### SQLでのトランザクション

```sql
BEGIN TRANSACTION;
  -- 複数の操作
  INSERT INTO messages (...) VALUES (...);
  UPDATE stats SET count = count + 1;
  DELETE FROM old_messages WHERE created_at < '2024-01-01';
COMMIT;  -- 全部成功したらコミット
```

エラーが起きたら：

```sql
ROLLBACK;  -- すべての変更を取り消す
```

---

## 4. Node.jsでのトランザクション実装

### better-sqlite3でのトランザクション

```javascript
const Database = require('better-sqlite3');
const db = new Database('data.db');

// トランザクション内で複数の操作を実行
function saveMessageWithStats(userId, username, content) {
  // トランザクション開始
  const transaction = db.transaction(() => {
    // 操作1: メッセージを保存
    const insertMsg = db.prepare(`
      INSERT INTO messages (user_id, username, content, created_at)
      VALUES (?, ?, ?, ?)
    `);
    const info = insertMsg.run(userId, username, content, new Date().toISOString());
    
    // 操作2: 統計を更新
    const updateStats = db.prepare(`
      INSERT INTO stats (user_id, message_count)
      VALUES (?, 1)
      ON CONFLICT(user_id) DO UPDATE SET message_count = message_count + 1
    `);
    updateStats.run(userId);
    
    return info.lastInsertRowid;
  });
  
  // トランザクションを実行
  try {
    const messageId = transaction();
    return { success: true, messageId };
  } catch (error) {
    // エラー時は自動でROLLBACK
    return { success: false, error: error.message };
  }
}
```

### トランザクションの重要性

```javascript
// ❌ トランザクションなし
function badSave(userId, content) {
  db.prepare('INSERT INTO messages (user_id, content) VALUES (?, ?)').run(userId, content);
  db.prepare('UPDATE stats SET count = count + 1 WHERE user_id = ?').run(userId);
  // ↑ 2行目でエラーが起きると、メッセージだけ保存されて統計が更新されない
}

// ✅ トランザクションあり
function goodSave(userId, content) {
  const transaction = db.transaction(() => {
    db.prepare('INSERT INTO messages (user_id, content) VALUES (?, ?)').run(userId, content);
    db.prepare('UPDATE stats SET count = count + 1 WHERE user_id = ?').run(userId);
  });
  transaction();  // エラーが起きたら両方とも取り消される
}
```

---

## 5. 排他制御とロック

### ロックとは？

ロックは、**「今、このデータを使っているから待って」という仕組み**です。

### レストランの例

```
【ロックなし】
客A: テーブル10の予約を確認 → 空いてる
客B: テーブル10の予約を確認 → 空いてる
客A: テーブル10を予約
客B: テーブル10を予約 ← 重複予約！

【ロックあり】
客A: テーブル10をロック → 予約確認 → 予約 → ロック解除
客B: テーブル10をロック待ち...
      ↓ Aが終わるまで待つ
      ロック取得 → 予約確認 → すでに予約済み → ロック解除
```

### SQLiteのロック機構

better-sqlite3は、**デフォルトで適切なロック**を行います。

```javascript
// better-sqlite3は自動でロック管理
const stmt = db.prepare('INSERT INTO messages (content) VALUES (?)');
stmt.run('こんにちは');  // ← 内部で自動ロック
```

### 注意が必要なケース

```javascript
// ❌ 複数のデータベース接続を開く（危険）
const db1 = new Database('data.db');
const db2 = new Database('data.db');  // 同じファイルに2つの接続

// db1とdb2が同時に書き込むと競合する可能性がある
```

**推奨**: データベース接続は1つだけ作り、使い回す

---

## 6. データベースが壊れるパターン集

### パターン1: プロセス終了時の未保存データ

```javascript
// ❌ 危険なコード
function saveMessage(content) {
  db.prepare('INSERT INTO messages (content) VALUES (?)').run(content);
  // ここでプロセスがKILLされると...
}

process.exit(0);  // 強制終了
```

**問題**: 書き込み中にプロセスが終了すると、データが壊れる可能性

**解決策**: 適切にクローズする

```javascript
// ✅ 安全なコード
function gracefulShutdown() {
  console.log('シャットダウン開始...');
  db.close();  // データベース接続を閉じる
  console.log('データベース接続を閉じました');
  process.exit(0);
}

// Ctrl+C でシャットダウン
process.on('SIGINT', gracefulShutdown);
```

### パターン2: 同時書き込みでのデッドロック

```javascript
// ❌ 危険なコード（デッドロックの可能性）
async function updateTwoTables(userId, content) {
  // 処理A: テーブル1 → テーブル2 の順で更新
  db.prepare('UPDATE table1 SET value = ? WHERE user_id = ?').run(content, userId);
  await someAsyncOperation();  // ← この間に処理Bが始まる
  db.prepare('UPDATE table2 SET value = ? WHERE user_id = ?').run(content, userId);
  
  // 処理B: テーブル2 → テーブル1 の順で更新
  // → デッドロックの可能性
}
```

**解決策**: トランザクションで1つの操作にまとめる

```javascript
// ✅ 安全なコード
function updateTwoTables(userId, content) {
  const transaction = db.transaction(() => {
    db.prepare('UPDATE table1 SET value = ? WHERE user_id = ?').run(content, userId);
    db.prepare('UPDATE table2 SET value = ? WHERE user_id = ?').run(content, userId);
  });
  transaction();
}
```

### パターン3: 非同期処理中のDB接続クローズ

```javascript
// ❌ 危険なコード
async function badAsyncSave(content) {
  const stmt = db.prepare('INSERT INTO messages (content) VALUES (?)');
  
  setTimeout(() => {
    stmt.run(content);  // ← この時点でdbがcloseされている可能性
  }, 1000);
  
  db.close();  // すぐにクローズ
}
```

**解決策**: すべての操作が終わってからクローズ

```javascript
// ✅ 安全なコード
async function goodAsyncSave(content) {
  const stmt = db.prepare('INSERT INTO messages (content) VALUES (?)');
  
  await new Promise((resolve) => {
    setTimeout(() => {
      stmt.run(content);
      resolve();
    }, 1000);
  });
  
  // すべての操作が終わってからクローズ
}
```

---

## 7. 実践：安全なDB操作の実装

### プロジェクトフォルダの作成

```
my-discord-bot/
├── database-safe.js
├── test-concurrent.js
└── data.db
```

### ステップ1: database-safe.js（安全なDB操作）

**ファイル**: `my-discord-bot/database-safe.js`

```javascript
const Database = require('better-sqlite3');

// データベース接続（シングルトンパターン）
let db = null;

function getDatabase() {
  if (!db) {
    db = new Database('data.db');
    console.log('✅ データベース接続を確立');
  }
  return db;
}

// テーブル作成
function initDatabase() {
  const database = getDatabase();
  
  // messagesテーブル
  database.exec(`
    CREATE TABLE IF NOT EXISTS messages (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id TEXT NOT NULL,
      username TEXT NOT NULL,
      content TEXT NOT NULL,
      created_at TEXT NOT NULL
    )
  `);
  
  // statsテーブル
  database.exec(`
    CREATE TABLE IF NOT EXISTS stats (
      user_id TEXT PRIMARY KEY,
      message_count INTEGER NOT NULL DEFAULT 0,
      last_message_at TEXT
    )
  `);
  
  console.log('✅ テーブル作成完了');
}

// トランザクションを使った安全な保存
function saveMessageSafe(userId, username, content) {
  const database = getDatabase();
  
  const transaction = database.transaction(() => {
    // 1. メッセージを保存
    const insertMsg = database.prepare(`
      INSERT INTO messages (user_id, username, content, created_at)
      VALUES (?, ?, ?, ?)
    `);
    const now = new Date().toISOString();
    const info = insertMsg.run(userId, username, content, now);
    
    // 2. 統計を更新
    const upsertStats = database.prepare(`
      INSERT INTO stats (user_id, message_count, last_message_at)
      VALUES (?, 1, ?)
      ON CONFLICT(user_id) DO UPDATE SET
        message_count = message_count + 1,
        last_message_at = ?
    `);
    upsertStats.run(userId, now, now);
    
    return info.lastInsertRowid;
  });
  
  try {
    const messageId = transaction();
    return { success: true, messageId };
  } catch (error) {
    console.error('❌ 保存エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// メッセージと統計を取得
function getMessagesWithStats(userId) {
  const database = getDatabase();
  
  const messages = database.prepare(`
    SELECT * FROM messages WHERE user_id = ? ORDER BY created_at ASC
  `).all(userId);
  
  const stats = database.prepare(`
    SELECT * FROM stats WHERE user_id = ?
  `).get(userId);
  
  return { messages, stats };
}

// 統計情報を取得
function getAllStats() {
  const database = getDatabase();
  const stmt = database.prepare('SELECT * FROM stats ORDER BY message_count DESC');
  return stmt.all();
}

// 安全なクローズ
function closeDatabase() {
  if (db) {
    db.close();
    db = null;
    console.log('✅ データベース接続を閉じました');
  }
}

// プロセス終了時のクリーンアップ
process.on('SIGINT', () => {
  console.log('\n\n🛑 シャットダウン開始...');
  closeDatabase();
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('\n\n🛑 シャットダウン開始...');
  closeDatabase();
  process.exit(0);
});

// 外部に公開
module.exports = {
  initDatabase,
  saveMessageSafe,
  getMessagesWithStats,
  getAllStats,
  closeDatabase
};
```

### ステップ2: test-concurrent.js（同時アクセステスト）

**ファイル**: `my-discord-bot/test-concurrent.js`

```javascript
const db = require('./database-safe.js');

console.log("=== 同時アクセステスト ===\n");

// データベース初期化
db.initDatabase();
console.log("");

// 複数のユーザーが同時にメッセージを送信するシミュレーション
async function simulateConcurrentAccess() {
  console.log("【テスト1】10人のユーザーが同時にメッセージを送信\n");
  
  const users = [
    { id: '1', name: '太郎' },
    { id: '2', name: '花子' },
    { id: '3', name: '次郎' },
    { id: '4', name: '四郎' },
    { id: '5', name: '五郎' },
    { id: '6', name: '六郎' },
    { id: '7', name: '七郎' },
    { id: '8', name: '八郎' },
    { id: '9', name: '九郎' },
    { id: '10', name: '十郎' }
  ];
  
  // すべてのユーザーが同時にメッセージを送信
  const promises = users.map(user => {
    return new Promise((resolve) => {
      // ランダムな遅延（0〜10ms）
      const delay = Math.random() * 10;
      
      setTimeout(() => {
        const result = db.saveMessageSafe(
          user.id,
          user.name,
          `${user.name}からのメッセージ`
        );
        
        if (result.success) {
          console.log(`✅ ${user.name}: メッセージ保存成功 (ID: ${result.messageId})`);
        } else {
          console.log(`❌ ${user.name}: メッセージ保存失敗`);
        }
        
        resolve(result);
      }, delay);
    });
  });
  
  // すべての保存が完了するまで待つ
  const results = await Promise.all(promises);
  
  const successCount = results.filter(r => r.success).length;
  const failCount = results.filter(r => !r.success).length;
  
  console.log(`\n📊 結果: 成功 ${successCount}件, 失敗 ${failCount}件\n`);
}

// 同じユーザーが連続でメッセージを送信
async function simulateRapidMessages() {
  console.log("【テスト2】太郎が10回連続でメッセージを送信\n");
  
  const promises = [];
  for (let i = 1; i <= 10; i++) {
    promises.push(
      new Promise((resolve) => {
        setTimeout(() => {
          const result = db.saveMessageSafe('1', '太郎', `メッセージ${i}`);
          console.log(`✅ メッセージ${i}保存完了`);
          resolve(result);
        }, i * 5);  // 5ms間隔で送信
      })
    );
  }
  
  await Promise.all(promises);
  console.log("");
}

// 統計情報を表示
function showStats() {
  console.log("【テスト3】統計情報を表示\n");
  
  const stats = db.getAllStats();
  console.log("📊 ユーザー別メッセージ数:");
  stats.forEach(stat => {
    const time = new Date(stat.last_message_at).toLocaleTimeString();
    console.log(`  ${stat.user_id}: ${stat.message_count}件 (最終: ${time})`);
  });
  console.log("");
}

// メイン処理
async function runTests() {
  try {
    // テスト1: 同時アクセス
    await simulateConcurrentAccess();
    
    // テスト2: 連続送信
    await simulateRapidMessages();
    
    // テスト3: 統計表示
    showStats();
    
    // 太郎のメッセージを確認
    console.log("【テスト4】太郎のメッセージを確認\n");
    const taroData = db.getMessagesWithStats('1');
    console.log(`📊 太郎のメッセージ数: ${taroData.stats.message_count}件`);
    console.log(`📝 メッセージ一覧:`);
    taroData.messages.forEach((msg, index) => {
      console.log(`  ${index + 1}. ${msg.content}`);
    });
    
  } catch (error) {
    console.error('❌ テスト中にエラーが発生:', error);
  } finally {
    // データベース接続を閉じる
    console.log("\n");
    db.closeDatabase();
    console.log("\n=== テスト完了 ===");
  }
}

// テスト実行
runTests();
```

### 実行方法

```bash
# プロジェクトフォルダに移動
cd my-discord-bot

# 実行
node test-concurrent.js
```

### 実行結果

```
=== 同時アクセステスト ===

✅ データベース接続を確立
✅ テーブル作成完了

【テスト1】10人のユーザーが同時にメッセージを送信

✅ 八郎: メッセージ保存成功 (ID: 1)
✅ 三郎: メッセージ保存成功 (ID: 2)
✅ 五郎: メッセージ保存成功 (ID: 3)
✅ 太郎: メッセージ保存成功 (ID: 4)
✅ 花子: メッセージ保存成功 (ID: 5)
✅ 七郎: メッセージ保存成功 (ID: 6)
✅ 十郎: メッセージ保存成功 (ID: 7)
✅ 四郎: メッセージ保存成功 (ID: 8)
✅ 六郎: メッセージ保存成功 (ID: 9)
✅ 九郎: メッセージ保存成功 (ID: 10)

📊 結果: 成功 10件, 失敗 0件

【テスト2】太郎が10回連続でメッセージを送信

✅ メッセージ1保存完了
✅ メッセージ2保存完了
✅ メッセージ3保存完了
✅ メッセージ4保存完了
✅ メッセージ5保存完了
✅ メッセージ6保存完了
✅ メッセージ7保存完了
✅ メッセージ8保存完了
✅ メッセージ9保存完了
✅ メッセージ10保存完了

【テスト3】統計情報を表示

📊 ユーザー別メッセージ数:
  1: 11件 (最終: 10:30:15)
  2: 1件 (最終: 10:30:05)
  3: 1件 (最終: 10:30:05)
  ...

【テスト4】太郎のメッセージを確認

📊 太郎のメッセージ数: 11件
📝 メッセージ一覧:
  1. 太郎からのメッセージ
  2. メッセージ1
  3. メッセージ2
  ...


✅ データベース接続を閉じました

=== テスト完了 ===
```

---

## 8. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── database-safe.js
├── test-concurrent.js
└── data.db (自動生成)
```

### database-safe.js（完成版）

**ファイルの場所**: `my-discord-bot/database-safe.js`

```javascript
const Database = require('better-sqlite3');

// データベース接続（シングルトンパターン）
let db = null;

function getDatabase() {
  if (!db) {
    db = new Database('data.db');
    // WALモードを有効化（パフォーマンス向上）
    db.pragma('journal_mode = WAL');
    console.log('✅ データベース接続を確立');
  }
  return db;
}

// テーブル作成
function initDatabase() {
  const database = getDatabase();
  
  // messagesテーブル
  database.exec(`
    CREATE TABLE IF NOT EXISTS messages (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id TEXT NOT NULL,
      username TEXT NOT NULL,
      content TEXT NOT NULL,
      created_at TEXT NOT NULL
    )
  `);
  
  // statsテーブル
  database.exec(`
    CREATE TABLE IF NOT EXISTS stats (
      user_id TEXT PRIMARY KEY,
      message_count INTEGER NOT NULL DEFAULT 0,
      last_message_at TEXT
    )
  `);
  
  // インデックス作成（検索高速化）
  database.exec(`
    CREATE INDEX IF NOT EXISTS idx_messages_user_id ON messages(user_id)
  `);
  
  database.exec(`
    CREATE INDEX IF NOT EXISTS idx_messages_created_at ON messages(created_at)
  `);
  
  console.log('✅ テーブル作成完了');
}

// トランザクションを使った安全な保存
function saveMessageSafe(userId, username, content) {
  const database = getDatabase();
  
  const transaction = database.transaction(() => {
    // 1. メッセージを保存
    const insertMsg = database.prepare(`
      INSERT INTO messages (user_id, username, content, created_at)
      VALUES (?, ?, ?, ?)
    `);
    const now = new Date().toISOString();
    const info = insertMsg.run(userId, username, content, now);
    
    // 2. 統計を更新（UPSERT: 存在すれば更新、なければ挿入）
    const upsertStats = database.prepare(`
      INSERT INTO stats (user_id, message_count, last_message_at)
      VALUES (?, 1, ?)
      ON CONFLICT(user_id) DO UPDATE SET
        message_count = message_count + 1,
        last_message_at = ?
    `);
    upsertStats.run(userId, now, now);
    
    return info.lastInsertRowid;
  });
  
  try {
    const messageId = transaction();
    return { success: true, messageId };
  } catch (error) {
    console.error('❌ 保存エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// メッセージと統計を取得
function getMessagesWithStats(userId) {
  const database = getDatabase();
  
  const messages = database.prepare(`
    SELECT * FROM messages WHERE user_id = ? ORDER BY created_at ASC
  `).all(userId);
  
  const stats = database.prepare(`
    SELECT * FROM stats WHERE user_id = ?
  `).get(userId);
  
  return { messages, stats: stats || { message_count: 0 } };
}

// 統計情報を取得
function getAllStats() {
  const database = getDatabase();
  const stmt = database.prepare('SELECT * FROM stats ORDER BY message_count DESC');
  return stmt.all();
}

// 古いメッセージを削除（トランザクション使用）
function deleteOldMessages(daysOld) {
  const database = getDatabase();
  
  const transaction = database.transaction(() => {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysOld);
    const cutoffStr = cutoffDate.toISOString();
    
    const stmt = database.prepare('DELETE FROM messages WHERE created_at < ?');
    const info = stmt.run(cutoffStr);
    
    return info.changes;
  });
  
  try {
    const deletedCount = transaction();
    return { success: true, deletedCount };
  } catch (error) {
    console.error('❌ 削除エラー:', error.message);
    return { success: false, error: error.message };
  }
}

// 安全なクローズ
function closeDatabase() {
  if (db) {
    db.close();
    db = null;
    console.log('✅ データベース接続を閉じました');
  }
}

// プロセス終了時のクリーンアップ
process.on('SIGINT', () => {
  console.log('\n\n🛑 シャットダウン開始...');
  closeDatabase();
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('\n\n🛑 シャットダウン開始...');
  closeDatabase();
  process.exit(0);
});

// 外部に公開
module.exports = {
  initDatabase,
  saveMessageSafe,
  getMessagesWithStats,
  getAllStats,
  deleteOldMessages,
  closeDatabase
};
```

### test-concurrent.js（完成版）

**上記の「ステップ2」のコードと同じ**

---

## 9. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] 同時アクセスでIDが重複する危険性を理解した
- [ ] AUTOINCREMENTで自動採番できる
- [ ] トランザクションは「全部成功 or 全部失敗」にする仕組み
- [ ] ロックは「今使っているから待って」という仕組み
- [ ] データベース接続はシングルトンパターンで1つだけ作る
- [ ] プロセス終了時は必ずcloseする

### データベースが壊れる3大原因

1. **同時書き込み**: トランザクションで防ぐ
2. **書き込み中の強制終了**: 適切なクローズ処理で防ぐ
3. **複数の接続**: シングルトンパターンで防ぐ

### 次回予告

**第8回: 外部APIとGemini APIの理解**

次回は、「外部APIとの通信」を学びます。

- APIとHTTPリクエストの本質
- Gemini APIの仕組み
- APIキーの秘匿方法
- エラーハンドリング

---

## 10. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: トランザクションはなぜ必要？

<details>
<summary>答えの例</summary>

複数の操作を「全部成功 or 全部失敗」にするため。  
例えば、メッセージ保存と統計更新の2つの操作があるとき、片方だけ成功すると整合性が崩れる。  
トランザクションを使えば、どちらかがエラーになったら両方とも取り消される。

</details>

### Q2: なぜデータベース接続を1つだけにするの？

<details>
<summary>答えの例</summary>

同じファイルに複数の接続を開くと、ロックの競合が起きやすくなる。  
シングルトンパターンで接続を1つだけにすれば、すべての操作が同じ接続を通るので、ロックが正しく機能する。

</details>

### Q3: WALモードとは？

<details>
<summary>答えの例</summary>

Write-Ahead Loggingの略で、書き込みを高速化する仕組み。  
通常モードでは、書き込み中は読み込みもブロックされるが、WALモードでは読み込みと書き込みを同時に行える。  
パフォーマンスが向上し、同時アクセスにも強くなる。

</details>

---

これらの質問に答えられれば、第7回は完璧です！

次回、[第8回: 外部APIとGemini APIの理解](./lesson08.md) でお会いしましょう。

---

**学習メモ**: トランザクションは強力ですが、必要以上に長く保持すると他の処理がブロックされます。必要最小限の操作だけをトランザクションに含めましょう。
