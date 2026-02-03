# 第9回: セキュリティ・エラー・安全設計

## この回で学ぶこと

- try/catchの真意（エラーを「握りつぶす」のではなく「制御する」）
- バリデーション（入力値チェック）の重要性
- APIキー流出経路の全パターン
- スパム対策とレート制限
- 安全なBot設計の原則

## ⚠️ 重要度：最高

この回は、**あなたのBotを守るための知識**です。  
セキュリティを軽視すると、Bot が悪用されたり、高額請求が来たりします。

## ゴール

この回を終えたあなたは、**危険な実装にブレーキをかけられる**ようになります。

---

## 1. try/catchの真意

### try/catchの誤解

多くの初心者は、try/catchを「エラーを消す魔法」だと誤解します。

```javascript
// ❌ 間違った使い方
try {
  someDangerousFunction();
} catch (error) {
  // 何もしない = エラーを握りつぶす
}
```

**これは最悪のパターン**

### try/catchの本当の役割

try/catchは、**エラーを制御し、適切に対処する**ための仕組みです。

```javascript
// ✅ 正しい使い方
try {
  const result = await saveToDatabase(data);
  return { success: true, result };
} catch (error) {
  // 1. ログに記録
  console.error('データベース保存エラー:', error.message);
  
  // 2. ユーザーに通知
  await message.reply('エラーが発生しました。しばらくしてからお試しください。');
  
  // 3. エラー情報を返す
  return { success: false, error: error.message };
}
```

### エラーハンドリングの3原則

1. **ログに記録する**（原因を追跡できるように）
2. **ユーザーに通知する**（何が起きたか分かるように）
3. **適切に処理する**（リトライ、代替処理、または終了）

---

## 2. バリデーション（入力値チェック）

### バリデーションとは？

バリデーションは、**ユーザーの入力が正しいかチェックする**処理です。

### なぜバリデーションが必要なのか？

```javascript
// ❌ バリデーションなし
function saveMessage(userId, content) {
  db.prepare('INSERT INTO messages (user_id, content) VALUES (?, ?)').run(userId, content);
}

// ユーザーが空文字を送ったら？
saveMessage('123', '');  // 空のメッセージが保存される

// ユーザーが10万文字送ったら？
saveMessage('123', 'あ'.repeat(100000));  // DBが肥大化
```

### バリデーションの実装

```javascript
// ✅ バリデーションあり
function saveMessage(userId, content) {
  // 入力値チェック
  if (!userId || typeof userId !== 'string') {
    throw new Error('user_idが不正です');
  }
  
  if (!content || typeof content !== 'string') {
    throw new Error('contentが不正です');
  }
  
  if (content.length === 0) {
    throw new Error('空のメッセージは保存できません');
  }
  
  if (content.length > 2000) {
    throw new Error('メッセージが長すぎます（2000文字まで）');
  }
  
  // バリデーション通過後に保存
  db.prepare('INSERT INTO messages (user_id, content) VALUES (?, ?)').run(userId, content);
}
```

### Discord Botでのバリデーション例

```javascript
function handleMessage(message) {
  // Bot自身のメッセージを無視
  if (message.author.bot) {
    return;
  }
  
  // メッセージが空
  if (!message.content || message.content.trim().length === 0) {
    return;
  }
  
  // メッセージが長すぎる
  if (message.content.length > 2000) {
    message.reply('メッセージが長すぎます（2000文字以内にしてください）');
    return;
  }
  
  // 処理を続行
  processMessage(message);
}
```

---

## 3. APIキー流出経路の全パターン

### 流出経路1: コードに直接書く

```javascript
// ❌ 最悪のパターン
const apiKey = "AIzaSyD1234567890abcdefghijklmnopqrstuvwxyz";
```

**対策**: .envファイルで管理

### 流出経路2: Gitにコミット

```bash
# ❌ .envファイルをコミット
git add .env
git commit -m "環境変数追加"
git push
```

**対策**: .gitignoreに.envを追加

### 流出経路3: スクリーンショット

```
例: 質問掲示板に「エラーが出ます」とスクリーンショットを投稿
→ 画像にAPIキーが写り込んでいる
```

**対策**: スクリーンショットを撮る前に、APIキーを隠す

### 流出経路4: ログ出力

```javascript
// ❌ APIキーをログに出力
console.log('APIキー:', process.env.GEMINI_API_KEY);
```

**対策**: APIキーは絶対にログに出力しない

### 流出経路5: エラーメッセージ

```javascript
// ❌ エラーメッセージにAPIキーが含まれる
const apiUrl = `https://api.example.com?key=${apiKey}`;
console.error('エラーURL:', apiUrl);
```

**対策**: エラーメッセージにAPIキーを含めない

### APIキーを守るチェックリスト

- [ ] コードに直接書いていない
- [ ] .envファイルで管理している
- [ ] .gitignoreに.envを追加している
- [ ] スクリーンショットに写り込んでいない
- [ ] ログに出力していない
- [ ] エラーメッセージに含まれていない

---

## 4. スパム対策とレート制限

### スパム攻撃とは？

スパム攻撃は、**意図的に大量のリクエストを送ってBotを妨害する**行為です。

```
例: 悪意のあるユーザーが1秒間に100回メッセージを送信
→ Botが処理しきれず、他のユーザーが使えなくなる
→ API呼び出しが大量に発生し、高額請求
```

### 対策1: クールダウン（連続送信の制限）

```javascript
const cooldowns = new Map();
const COOLDOWN_TIME = 3000;  // 3秒

function handleMessage(message) {
  const userId = message.author.id;
  const now = Date.now();
  
  // クールダウンチェック
  if (cooldowns.has(userId)) {
    const lastTime = cooldowns.get(userId);
    const timePassed = now - lastTime;
    
    if (timePassed < COOLDOWN_TIME) {
      const remaining = Math.ceil((COOLDOWN_TIME - timePassed) / 1000);
      message.reply(`クールダウン中です。あと${remaining}秒お待ちください。`);
      return;
    }
  }
  
  // クールダウンを更新
  cooldowns.set(userId, now);
  
  // 処理を続行
  processMessage(message);
}
```

### 対策2: メッセージ長制限

```javascript
const MAX_MESSAGE_LENGTH = 2000;

function handleMessage(message) {
  if (message.content.length > MAX_MESSAGE_LENGTH) {
    message.reply(`メッセージは${MAX_MESSAGE_LENGTH}文字以内にしてください。`);
    return;
  }
  
  processMessage(message);
}
```

### 対策3: 特定ユーザーのブロック

```javascript
const BLOCKED_USERS = ['悪意のあるユーザーのID'];

function handleMessage(message) {
  if (BLOCKED_USERS.includes(message.author.id)) {
    console.log(`ブロックされたユーザー: ${message.author.id}`);
    return;
  }
  
  processMessage(message);
}
```

---

## 5. 安全なBot設計の原則

### 原則1: 最小権限の原則

**Botには必要最小限の権限だけを与える**

```
例: メッセージを読み書きするだけのBot
✅ 必要な権限: Read Messages, Send Messages
❌ 不要な権限: Manage Server, Kick Members
```

### 原則2: フェイルセーフ

**エラーが起きてもBotが停止しない設計**

```javascript
// ✅ フェイルセーフな設計
client.on('messageCreate', async (message) => {
  try {
    await handleMessage(message);
  } catch (error) {
    console.error('メッセージ処理エラー:', error);
    // エラーが起きてもBotは動き続ける
  }
});
```

### 原則3: ログとモニタリング

**エラーやアクセスをログに記録し、異常を検知**

```javascript
function logEvent(eventType, userId, details) {
  const timestamp = new Date().toISOString();
  console.log(`[${timestamp}] ${eventType} - User: ${userId} - ${details}`);
}

function handleMessage(message) {
  logEvent('MESSAGE', message.author.id, `Content: ${message.content.substring(0, 50)}`);
  
  try {
    processMessage(message);
  } catch (error) {
    logEvent('ERROR', message.author.id, error.message);
  }
}
```

### 原則4: データの暗号化

**機密データは暗号化して保存**

```javascript
const crypto = require('crypto');

// データを暗号化
function encrypt(text, key) {
  const cipher = crypto.createCipher('aes-256-cbc', key);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return encrypted;
}

// データを復号化
function decrypt(encrypted, key) {
  const decipher = crypto.createDecipher('aes-256-cbc', key);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

---

## 6. 実践：安全なBot実装

### プロジェクトフォルダの作成

```
my-discord-bot/
├── .env
├── .gitignore
├── security.js
└── test-security.js
```

### ステップ1: security.js（セキュリティ機能）

**ファイル**: `my-discord-bot/security.js`

```javascript
// クールダウン管理
class CooldownManager {
  constructor(cooldownTime = 3000) {
    this.cooldowns = new Map();
    this.cooldownTime = cooldownTime;
  }
  
  // クールダウンチェック
  check(userId) {
    const now = Date.now();
    
    if (this.cooldowns.has(userId)) {
      const lastTime = this.cooldowns.get(userId);
      const timePassed = now - lastTime;
      
      if (timePassed < this.cooldownTime) {
        const remaining = Math.ceil((this.cooldownTime - timePassed) / 1000);
        return {
          allowed: false,
          remaining: remaining
        };
      }
    }
    
    // クールダウンを更新
    this.cooldowns.set(userId, now);
    return { allowed: true };
  }
  
  // クールダウンをリセット
  reset(userId) {
    this.cooldowns.delete(userId);
  }
  
  // 定期的にクールダウンをクリーンアップ
  cleanup() {
    const now = Date.now();
    for (const [userId, lastTime] of this.cooldowns.entries()) {
      if (now - lastTime > this.cooldownTime * 10) {
        this.cooldowns.delete(userId);
      }
    }
  }
}

// バリデーション関数
class Validator {
  // メッセージ内容をバリデーション
  static validateMessage(content) {
    const errors = [];
    
    // 空チェック
    if (!content || content.trim().length === 0) {
      errors.push('メッセージが空です');
    }
    
    // 長さチェック
    if (content.length > 2000) {
      errors.push('メッセージが長すぎます（2000文字まで）');
    }
    
    // 不正文字チェック
    if (/[\x00-\x08\x0B-\x0C\x0E-\x1F]/.test(content)) {
      errors.push('不正な制御文字が含まれています');
    }
    
    return {
      valid: errors.length === 0,
      errors: errors
    };
  }
  
  // ユーザーIDをバリデーション
  static validateUserId(userId) {
    if (!userId || typeof userId !== 'string') {
      return { valid: false, error: 'ユーザーIDが不正です' };
    }
    
    if (!/^\d+$/.test(userId)) {
      return { valid: false, error: 'ユーザーIDは数字のみです' };
    }
    
    return { valid: true };
  }
}

// ブロックリスト管理
class BlockList {
  constructor() {
    this.blockedUsers = new Set();
    this.blockedWords = [];
  }
  
  // ユーザーをブロック
  blockUser(userId) {
    this.blockedUsers.add(userId);
  }
  
  // ユーザーのブロックを解除
  unblockUser(userId) {
    this.blockedUsers.delete(userId);
  }
  
  // ユーザーがブロックされているかチェック
  isUserBlocked(userId) {
    return this.blockedUsers.has(userId);
  }
  
  // NGワードを追加
  addBlockedWord(word) {
    this.blockedWords.push(word.toLowerCase());
  }
  
  // メッセージにNGワードが含まれているかチェック
  containsBlockedWord(message) {
    const lowerMessage = message.toLowerCase();
    for (const word of this.blockedWords) {
      if (lowerMessage.includes(word)) {
        return { blocked: true, word: word };
      }
    }
    return { blocked: false };
  }
}

// ログ管理
class Logger {
  static log(level, userId, message) {
    const timestamp = new Date().toISOString();
    const logMessage = `[${timestamp}] [${level}] User:${userId} - ${message}`;
    
    if (level === 'ERROR') {
      console.error(logMessage);
    } else if (level === 'WARN') {
      console.warn(logMessage);
    } else {
      console.log(logMessage);
    }
  }
  
  static info(userId, message) {
    this.log('INFO', userId, message);
  }
  
  static warn(userId, message) {
    this.log('WARN', userId, message);
  }
  
  static error(userId, message) {
    this.log('ERROR', userId, message);
  }
}

// 外部に公開
module.exports = {
  CooldownManager,
  Validator,
  BlockList,
  Logger
};
```

### ステップ2: test-security.js（テスト）

**ファイル**: `my-discord-bot/test-security.js`

```javascript
const { CooldownManager, Validator, BlockList, Logger } = require('./security.js');

console.log("=== セキュリティ機能テスト ===\n");

// テスト1: クールダウン
function testCooldown() {
  console.log("【テスト1】クールダウン機能\n");
  
  const cooldown = new CooldownManager(2000);  // 2秒
  const userId = '123456';
  
  // 1回目
  let result = cooldown.check(userId);
  console.log('1回目:', result.allowed ? '✅ 許可' : '❌ 拒否');
  
  // 2回目（すぐ）
  result = cooldown.check(userId);
  console.log('2回目（すぐ）:', result.allowed ? '✅ 許可' : `❌ 拒否（あと${result.remaining}秒）`);
  
  // 3回目（2秒後）
  setTimeout(() => {
    result = cooldown.check(userId);
    console.log('3回目（2秒後）:', result.allowed ? '✅ 許可' : '❌ 拒否');
    console.log("");
    
    testValidation();
  }, 2100);
}

// テスト2: バリデーション
function testValidation() {
  console.log("【テスト2】バリデーション機能\n");
  
  // 正常なメッセージ
  let result = Validator.validateMessage('こんにちは');
  console.log('正常なメッセージ:', result.valid ? '✅ 有効' : '❌ 無効');
  
  // 空のメッセージ
  result = Validator.validateMessage('');
  console.log('空のメッセージ:', result.valid ? '✅ 有効' : `❌ 無効 (${result.errors.join(', ')})`);
  
  // 長すぎるメッセージ
  result = Validator.validateMessage('あ'.repeat(2001));
  console.log('長すぎるメッセージ:', result.valid ? '✅ 有効' : `❌ 無効 (${result.errors.join(', ')})`);
  
  // ユーザーIDのバリデーション
  result = Validator.validateUserId('123456');
  console.log('正常なユーザーID:', result.valid ? '✅ 有効' : '❌ 無効');
  
  result = Validator.validateUserId('abc');
  console.log('不正なユーザーID:', result.valid ? '✅ 有効' : `❌ 無効 (${result.error})`);
  
  console.log("");
  testBlockList();
}

// テスト3: ブロックリスト
function testBlockList() {
  console.log("【テスト3】ブロックリスト機能\n");
  
  const blockList = new BlockList();
  
  // ユーザーをブロック
  blockList.blockUser('999999');
  console.log('ユーザー999999をブロック');
  
  console.log('ユーザー123456はブロック済み?', blockList.isUserBlocked('123456') ? 'はい' : 'いいえ');
  console.log('ユーザー999999はブロック済み?', blockList.isUserBlocked('999999') ? 'はい' : 'いいえ');
  
  // NGワードを追加
  blockList.addBlockedWord('spam');
  blockList.addBlockedWord('宣伝');
  console.log('\nNGワード追加: spam, 宣伝');
  
  let result = blockList.containsBlockedWord('こんにちは');
  console.log('メッセージ「こんにちは」:', result.blocked ? `❌ NG (${result.word})` : '✅ OK');
  
  result = blockList.containsBlockedWord('これはspamです');
  console.log('メッセージ「これはspamです」:', result.blocked ? `❌ NG (${result.word})` : '✅ OK');
  
  console.log("");
  testLogger();
}

// テスト4: ログ
function testLogger() {
  console.log("【テスト4】ログ機能\n");
  
  Logger.info('123456', 'メッセージを受信');
  Logger.warn('123456', 'クールダウン中');
  Logger.error('123456', 'API呼び出しエラー');
  
  console.log("\n=== すべてのテスト完了 ===");
}

// テスト実行
testCooldown();
```

### 実行方法

```bash
# プロジェクトフォルダに移動
cd my-discord-bot

# 実行
node test-security.js
```

### 実行結果

```
=== セキュリティ機能テスト ===

【テスト1】クールダウン機能

1回目: ✅ 許可
2回目（すぐ）: ❌ 拒否（あと2秒）
（2秒待機）
3回目（2秒後）: ✅ 許可

【テスト2】バリデーション機能

正常なメッセージ: ✅ 有効
空のメッセージ: ❌ 無効 (メッセージが空です)
長すぎるメッセージ: ❌ 無効 (メッセージが長すぎます（2000文字まで）)
正常なユーザーID: ✅ 有効
不正なユーザーID: ❌ 無効 (ユーザーIDは数字のみです)

【テスト3】ブロックリスト機能

ユーザー999999をブロック
ユーザー123456はブロック済み? いいえ
ユーザー999999はブロック済み? はい

NGワード追加: spam, 宣伝
メッセージ「こんにちは」: ✅ OK
メッセージ「これはspamです」: ❌ NG (spam)

【テスト4】ログ機能

[2024-01-01T10:00:00.000Z] [INFO] User:123456 - メッセージを受信
[2024-01-01T10:00:00.001Z] [WARN] User:123456 - クールダウン中
[2024-01-01T10:00:00.002Z] [ERROR] User:123456 - API呼び出しエラー

=== すべてのテスト完了 ===
```

---

## 7. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── security.js
└── test-security.js
```

### security.js（完成版）

**上記の「ステップ1」のコードと同じ**

### test-security.js（完成版）

**上記の「ステップ2」のコードと同じ**

---

## 8. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] try/catchは「エラーを制御する」ためのもの
- [ ] バリデーションで不正な入力を弾く
- [ ] APIキー流出経路を理解し、すべてふさぐ
- [ ] クールダウンでスパム攻撃を防ぐ
- [ ] ログとモニタリングで異常を検知
- [ ] 最小権限の原則を守る

### やってはいけないこと5選

1. **try/catchでエラーを握りつぶす**
2. **バリデーションなしでユーザー入力を保存**
3. **APIキーをコードに直接書く**
4. **クールダウンなしでAPI呼び出し**
5. **エラーをログに記録しない**

### 次回予告

**第10回: Discord bot実コード完全読解**

次回は、ついに**すべてを統合した完全版Bot**を作ります。

- Discordイベントの流れ
- 全要素の連携
- 実コードの精読
- 拡張ポイント

---

## 9. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: なぜバリデーションが重要なの？

<details>
<summary>答えの例</summary>

ユーザーの入力は、常に正しいとは限らない。  
空のメッセージ、長すぎるメッセージ、不正な文字が含まれる可能性がある。  
バリデーションなしで保存すると、DBが壊れたり、Botが停止したり、セキュリティ問題が起きる。

</details>

### Q2: クールダウンはどう実装する？

<details>
<summary>答えの例</summary>

ユーザーIDと最後のアクセス時刻をMapで管理。  
新しいリクエストが来たら、前回からの経過時間をチェック。  
クールダウン時間が経過していなければ拒否、経過していれば許可して時刻を更新。

</details>

### Q3: try/catchで「何もしない」のはなぜダメ？

<details>
<summary>答えの例</summary>

エラーが起きた原因が分からなくなる。  
ログに記録しないと、後から問題を追跡できない。  
ユーザーにも通知しないと、「なぜ動かないの？」と混乱させる。  
エラーは「制御する」ものであって、「隠す」ものではない。

</details>

---

これらの質問に答えられれば、第9回は完璧です！

次回、[第10回: Discord bot実コード完全読解](./lesson10.md) でお会いしましょう。

---

**学習メモ**: セキュリティは「後から追加する」ものではなく、「最初から設計に組み込む」ものです。
