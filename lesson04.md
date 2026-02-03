# 第4回: 条件分岐と繰り返し (Botが判断する理由)

## この回で学ぶこと

- if/else による条件分岐（Botの判断の仕組み）
- 比較演算子と論理演算子（条件の書き方）
- for/forEach による繰り返し処理（配列の全要素を処理）
- filter/map による配列操作（データの絞り込みと変換）
- コマンド判定の実装

## ゴール

この回を終えたあなたは、**反応条件を自分で変更できる**ようになります。

---

## 1. if文：条件分岐の基本

### if文の本質：「もし〜なら」の実装

if文は、**条件によって処理を変える**ための構文です。

```javascript
const age = 18;

if (age >= 18) {
  console.log("成人です");
}
```

### if-else：条件が偽の場合の処理

```javascript
const age = 15;

if (age >= 18) {
  console.log("成人です");
} else {
  console.log("未成年です");
}
```

### 実行結果

```
未成年です
```

### if-else if-else：複数の条件

```javascript
const score = 75;

if (score >= 90) {
  console.log("優秀です");
} else if (score >= 70) {
  console.log("合格です");
} else if (score >= 50) {
  console.log("もう少しです");
} else {
  console.log("不合格です");
}
```

### 実行結果

```
合格です
```

---

## 2. 比較演算子：条件の書き方

### 基本的な比較演算子

```javascript
const a = 10;
const b = 20;

console.log(a === b);  // false (等しい)
console.log(a !== b);  // true  (等しくない)
console.log(a < b);    // true  (より小さい)
console.log(a > b);    // false (より大きい)
console.log(a <= b);   // true  (以下)
console.log(a >= b);   // false (以上)
```

### === と == の違い（重要）

```javascript
const num = 5;
const str = "5";

console.log(num == str);   // true  (型変換して比較)
console.log(num === str);  // false (型も含めて厳密に比較)
```

**推奨**: 常に`===`と`!==`を使う（意図しない型変換を防ぐ）

### 文字列の比較

```javascript
const message = "こんにちは";

if (message === "こんにちは") {
  console.log("挨拶を受け取りました");
}

// 部分一致の確認
if (message.includes("こんにち")) {
  console.log("挨拶が含まれています");
}

// 前方一致の確認
if (message.startsWith("こんにち")) {
  console.log("挨拶で始まっています");
}
```

---

## 3. 論理演算子：複数の条件を組み合わせる

### && (AND)：すべての条件が真

```javascript
const age = 25;
const hasLicense = true;

if (age >= 18 && hasLicense) {
  console.log("運転できます");
}
```

### || (OR)：いずれかの条件が真

```javascript
const day = "土曜日";

if (day === "土曜日" || day === "日曜日") {
  console.log("週末です");
}
```

### ! (NOT)：条件の反転

```javascript
const isBot = false;

if (!isBot) {
  console.log("人間のメッセージです");
}
```

### 複雑な条件の組み合わせ

```javascript
const age = 20;
const isStudent = true;
const hasTicket = false;

if ((age < 18 || isStudent) && !hasTicket) {
  console.log("割引対象です");
}
```

---

## 4. Discord Botでの条件分岐

### Bot自身のメッセージを無視

```javascript
function handleMessage(message) {
  // Bot自身のメッセージなら何もしない
  if (message.author.bot) {
    return;
  }
  
  console.log("人間のメッセージを処理します");
}
```

### コマンド判定

```javascript
function handleMessage(message) {
  const content = message.content;
  
  if (content.startsWith("!help")) {
    console.log("ヘルプコマンドを実行");
  } else if (content.startsWith("!ping")) {
    console.log("Pingコマンドを実行");
  } else if (content.startsWith("!clear")) {
    console.log("クリアコマンドを実行");
  } else {
    console.log("通常のメッセージとして処理");
  }
}
```

### 権限チェック

```javascript
function handleAdminCommand(message) {
  const isAdmin = message.member.permissions.has("Administrator");
  
  if (!isAdmin) {
    message.reply("このコマンドは管理者のみ使用できます");
    return;
  }
  
  // 管理者コマンドの処理
  console.log("管理者コマンドを実行");
}
```

---

## 5. 繰り返し処理：for文

### for文の基本

```javascript
// 0から4まで繰り返す
for (let i = 0; i < 5; i++) {
  console.log(`カウント: ${i}`);
}
```

### 実行結果

```
カウント: 0
カウント: 1
カウント: 2
カウント: 3
カウント: 4
```

### for文の構造

```javascript
for (初期化; 条件; 更新) {
  // 繰り返す処理
}

// 具体例
for (let i = 0; i < 5; i++) {
  //   ↑      ↑     ↑
  //   初期化  条件  更新
}
```

### 配列をfor文で処理

```javascript
const fruits = ["りんご", "バナナ", "オレンジ"];

for (let i = 0; i < fruits.length; i++) {
  console.log(`${i + 1}番目: ${fruits[i]}`);
}
```

### 実行結果

```
1番目: りんご
2番目: バナナ
3番目: オレンジ
```

---

## 6. 繰り返し処理：forEach

### forEachの基本

forEachは、**配列の各要素に対して処理を実行する**メソッドです。

```javascript
const fruits = ["りんご", "バナナ", "オレンジ"];

fruits.forEach((fruit) => {
  console.log(fruit);
});
```

### 実行結果

```
りんご
バナナ
オレンジ
```

### インデックスも取得

```javascript
const fruits = ["りんご", "バナナ", "オレンジ"];

fruits.forEach((fruit, index) => {
  console.log(`${index + 1}番目: ${fruit}`);
});
```

### forとforEachの違い

```javascript
// for文: 明示的なカウンター
for (let i = 0; i < fruits.length; i++) {
  console.log(fruits[i]);
}

// forEach: 各要素を直接扱う
fruits.forEach((fruit) => {
  console.log(fruit);
});
```

**推奨**: 配列の全要素を処理するなら、forEachの方が読みやすい

---

## 7. 配列の操作：filter と map

### filter：条件に合う要素だけを取り出す

```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// 偶数だけを取り出す
const evenNumbers = numbers.filter((num) => {
  return num % 2 === 0;
});

console.log(evenNumbers);  // [2, 4, 6, 8, 10]
```

### Discord Botでのfilter活用例

```javascript
// 過去10件のメッセージから、ユーザーのメッセージだけを取り出す
const messages = [
  { author: { bot: false }, content: "こんにちは" },
  { author: { bot: true }, content: "Bot応答" },
  { author: { bot: false }, content: "ありがとう" },
  { author: { bot: true }, content: "Bot応答" }
];

const userMessages = messages.filter((msg) => {
  return !msg.author.bot;
});

console.log(userMessages.length);  // 2
```

### map：各要素を変換する

```javascript
const numbers = [1, 2, 3, 4, 5];

// 各要素を2倍にする
const doubled = numbers.map((num) => {
  return num * 2;
});

console.log(doubled);  // [2, 4, 6, 8, 10]
```

### Discord Botでのmap活用例

```javascript
const messages = [
  { author: { username: "太郎" }, content: "こんにちは" },
  { author: { username: "花子" }, content: "ありがとう" }
];

// メッセージを整形
const formatted = messages.map((msg) => {
  return `[${msg.author.username}] ${msg.content}`;
});

console.log(formatted);
// ["[太郎] こんにちは", "[花子] ありがとう"]
```

---

## 8. 実践：コマンド判定システムを作る

### プロジェクトフォルダの作成

```
my-discord-bot/
├── index.js
├── handlers/
│   └── commandHandler.js
└── services/
    └── commandService.js
```

### ステップ1: commandService.js（コマンド処理）

**ファイル**: `my-discord-bot/services/commandService.js`

```javascript
// ヘルプコマンドの処理
function executeHelp() {
  return `
📖 利用可能なコマンド:
!help   - このヘルプを表示
!ping   - Botの応答速度を確認
!clear  - 会話履歴をクリア
!info   - Botの情報を表示
  `.trim();
}

// Pingコマンドの処理
function executePing() {
  return "🏓 Pong! Bot は正常に動作しています。";
}

// クリアコマンドの処理
function executeClear(username) {
  return `🗑️ ${username}さんの会話履歴をクリアしました。`;
}

// 情報コマンドの処理
function executeInfo() {
  return `
ℹ️ Bot情報:
- 名前: MyBot
- バージョン: 1.0.0
- 開発者: あなた
  `.trim();
}

// 外部に公開
module.exports = {
  executeHelp,
  executePing,
  executeClear,
  executeInfo
};
```

### ステップ2: commandHandler.js（コマンド振り分け）

**ファイル**: `my-discord-bot/handlers/commandHandler.js`

```javascript
// commandService.js を読み込む
const commandService = require('../services/commandService.js');

// コマンドを判定して実行
function handleCommand(message) {
  const content = message.content;
  const username = message.author.username;

  // コマンド判定
  if (content === "!help") {
    const response = commandService.executeHelp();
    return response;
  } else if (content === "!ping") {
    const response = commandService.executePing();
    return response;
  } else if (content === "!clear") {
    const response = commandService.executeClear(username);
    return response;
  } else if (content === "!info") {
    const response = commandService.executeInfo();
    return response;
  } else if (content.startsWith("!")) {
    // 不明なコマンド
    return `❌ 不明なコマンドです。!help でヘルプを確認してください。`;
  } else {
    // コマンドではない通常のメッセージ
    return null;
  }
}

// コマンドかどうかを判定
function isCommand(message) {
  return message.content.startsWith("!");
}

// 外部に公開
module.exports = {
  handleCommand,
  isCommand
};
```

### ステップ3: index.js（起動ファイル）

**ファイル**: `my-discord-bot/index.js`

```javascript
// commandHandler.js を読み込む
const commandHandler = require('./handlers/commandHandler.js');

// テスト用のメッセージ
const testMessages = [
  { author: { username: "太郎" }, content: "!help" },
  { author: { username: "花子" }, content: "!ping" },
  { author: { username: "次郎" }, content: "!clear" },
  { author: { username: "四郎" }, content: "!info" },
  { author: { username: "五郎" }, content: "!unknown" },
  { author: { username: "六郎" }, content: "こんにちは" }
];

// テスト実行
console.log("=== コマンド判定テスト ===\n");

testMessages.forEach((message, index) => {
  console.log(`--- テスト ${index + 1} ---`);
  console.log(`入力: [${message.author.username}] ${message.content}`);
  
  if (commandHandler.isCommand(message)) {
    const response = commandHandler.handleCommand(message);
    console.log(`Bot: ${response}`);
  } else {
    console.log("Bot: (コマンドではないので通常処理)");
  }
  
  console.log("");
});
```

### 実行方法

```bash
# プロジェクトフォルダに移動
cd my-discord-bot

# 実行
node index.js
```

### 実行結果

```
=== コマンド判定テスト ===

--- テスト 1 ---
入力: [太郎] !help
Bot: 📖 利用可能なコマンド:
!help   - このヘルプを表示
!ping   - Botの応答速度を確認
!clear  - 会話履歴をクリア
!info   - Botの情報を表示

--- テスト 2 ---
入力: [花子] !ping
Bot: 🏓 Pong! Bot は正常に動作しています。

--- テスト 3 ---
入力: [次郎] !clear
Bot: 🗑️ 次郎さんの会話履歴をクリアしました。

--- テスト 4 ---
入力: [四郎] !info
Bot: ℹ️ Bot情報:
- 名前: MyBot
- バージョン: 1.0.0
- 開発者: あなた

--- テスト 5 ---
入力: [五郎] !unknown
Bot: ❌ 不明なコマンドです。!help でヘルプを確認してください。

--- テスト 6 ---
入力: [六郎] こんにちは
Bot: (コマンドではないので通常処理)
```

---

## 9. 履歴フィルタリングの実装

### 過去のメッセージから必要な情報だけを取り出す

**ファイル**: `my-discord-bot/services/historyService.js`

```javascript
// 会話履歴から最新N件を取得
function getRecentMessages(messages, count) {
  // 配列の最後からcount件を取得
  return messages.slice(-count);
}

// Bot以外のメッセージだけを取得
function getUserMessages(messages) {
  return messages.filter((msg) => {
    return !msg.author.bot;
  });
}

// メッセージを整形して文字列に変換
function formatHistory(messages) {
  return messages.map((msg, index) => {
    const username = msg.author.username;
    const content = msg.content;
    return `${index + 1}. [${username}] ${content}`;
  }).join('\n');
}

// テスト用のサンプルデータ
const sampleMessages = [
  { author: { bot: false, username: "太郎" }, content: "こんにちは" },
  { author: { bot: true, username: "Bot" }, content: "こんにちは！" },
  { author: { bot: false, username: "太郎" }, content: "元気？" },
  { author: { bot: true, username: "Bot" }, content: "はい、元気です" },
  { author: { bot: false, username: "花子" }, content: "私も元気" },
  { author: { bot: true, username: "Bot" }, content: "良かったです" }
];

// テスト実行
console.log("=== 履歴フィルタリングテスト ===\n");

console.log("1. 最新3件を取得:");
const recent = getRecentMessages(sampleMessages, 3);
console.log(formatHistory(recent));
console.log("");

console.log("2. ユーザーメッセージのみ取得:");
const userOnly = getUserMessages(sampleMessages);
console.log(formatHistory(userOnly));
console.log("");

console.log("3. ユーザーメッセージの最新2件:");
const recentUserMessages = getRecentMessages(getUserMessages(sampleMessages), 2);
console.log(formatHistory(recentUserMessages));

// 外部に公開
module.exports = {
  getRecentMessages,
  getUserMessages,
  formatHistory
};
```

---

## 10. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── index.js
├── handlers/
│   └── commandHandler.js
└── services/
    ├── commandService.js
    └── historyService.js
```

### commandService.js（完成版）

**ファイルの場所**: `my-discord-bot/services/commandService.js`

```javascript
// ヘルプコマンドの処理
function executeHelp() {
  return `
📖 利用可能なコマンド:
!help   - このヘルプを表示
!ping   - Botの応答速度を確認
!clear  - 会話履歴をクリア
!info   - Botの情報を表示
  `.trim();
}

// Pingコマンドの処理
function executePing() {
  const timestamp = Date.now();
  return `🏓 Pong! Bot は正常に動作しています。(${timestamp})`;
}

// クリアコマンドの処理
function executeClear(username) {
  return `🗑️ ${username}さんの会話履歴をクリアしました。`;
}

// 情報コマンドの処理
function executeInfo() {
  return `
ℹ️ Bot情報:
- 名前: MyBot
- バージョン: 1.0.0
- 開発者: あなた
- 対応コマンド: 4個
  `.trim();
}

// 外部に公開
module.exports = {
  executeHelp,
  executePing,
  executeClear,
  executeInfo
};
```

### commandHandler.js（完成版）

**ファイルの場所**: `my-discord-bot/handlers/commandHandler.js`

```javascript
// commandService.js を読み込む
const commandService = require('../services/commandService.js');

// コマンドを判定して実行
function handleCommand(message) {
  const content = message.content.trim();
  const username = message.author.username;

  // コマンド判定（大文字小文字を区別しない）
  const command = content.toLowerCase();

  if (command === "!help") {
    return commandService.executeHelp();
  } else if (command === "!ping") {
    return commandService.executePing();
  } else if (command === "!clear") {
    return commandService.executeClear(username);
  } else if (command === "!info") {
    return commandService.executeInfo();
  } else if (content.startsWith("!")) {
    // 不明なコマンド
    return `❌ 不明なコマンド「${content}」です。!help でヘルプを確認してください。`;
  } else {
    // コマンドではない通常のメッセージ
    return null;
  }
}

// コマンドかどうかを判定
function isCommand(message) {
  return message.content.trim().startsWith("!");
}

// 外部に公開
module.exports = {
  handleCommand,
  isCommand
};
```

### historyService.js（完成版）

**ファイルの場所**: `my-discord-bot/services/historyService.js`

```javascript
// 会話履歴から最新N件を取得
function getRecentMessages(messages, count) {
  if (messages.length <= count) {
    return messages;
  }
  return messages.slice(-count);
}

// Bot以外のメッセージだけを取得
function getUserMessages(messages) {
  return messages.filter((msg) => !msg.author.bot);
}

// メッセージを整形して文字列に変換
function formatHistory(messages) {
  if (messages.length === 0) {
    return "(メッセージなし)";
  }
  
  return messages.map((msg, index) => {
    const username = msg.author.username;
    const content = msg.content;
    const timestamp = msg.timestamp || "不明";
    return `${index + 1}. [${timestamp}] ${username}: ${content}`;
  }).join('\n');
}

// メッセージ数をカウント
function countMessages(messages) {
  return {
    total: messages.length,
    user: messages.filter((msg) => !msg.author.bot).length,
    bot: messages.filter((msg) => msg.author.bot).length
  };
}

// 外部に公開
module.exports = {
  getRecentMessages,
  getUserMessages,
  formatHistory,
  countMessages
};
```

### index.js（完成版）

**ファイルの場所**: `my-discord-bot/index.js`

```javascript
// ハンドラとサービスを読み込む
const commandHandler = require('./handlers/commandHandler.js');
const historyService = require('./services/historyService.js');

// テスト用のメッセージ
const testMessages = [
  { author: { username: "太郎" }, content: "!help" },
  { author: { username: "花子" }, content: "!ping" },
  { author: { username: "次郎" }, content: "!CLEAR" },  // 大文字でも動作
  { author: { username: "四郎" }, content: "!info" },
  { author: { username: "五郎" }, content: "!unknown" },
  { author: { username: "六郎" }, content: "こんにちは" }
];

// コマンドテスト
console.log("=== コマンド判定テスト ===\n");

testMessages.forEach((message, index) => {
  console.log(`--- テスト ${index + 1} ---`);
  console.log(`入力: [${message.author.username}] ${message.content}`);
  
  if (commandHandler.isCommand(message)) {
    const response = commandHandler.handleCommand(message);
    console.log(`Bot: ${response}`);
  } else {
    console.log("Bot: (コマンドではないので通常処理)");
  }
  
  console.log("");
});

// 履歴フィルタリングテスト
const sampleHistory = [
  { author: { bot: false, username: "太郎" }, content: "こんにちは", timestamp: "10:00" },
  { author: { bot: true, username: "Bot" }, content: "こんにちは！", timestamp: "10:00" },
  { author: { bot: false, username: "太郎" }, content: "元気？", timestamp: "10:01" },
  { author: { bot: true, username: "Bot" }, content: "はい、元気です", timestamp: "10:01" },
  { author: { bot: false, username: "花子" }, content: "私も元気", timestamp: "10:02" },
  { author: { bot: true, username: "Bot" }, content: "良かったです", timestamp: "10:02" }
];

console.log("=== 履歴フィルタリングテスト ===\n");

console.log("1. 全メッセージ:");
console.log(historyService.formatHistory(sampleHistory));
console.log("");

console.log("2. ユーザーメッセージのみ:");
const userOnly = historyService.getUserMessages(sampleHistory);
console.log(historyService.formatHistory(userOnly));
console.log("");

console.log("3. 最新3件:");
const recent = historyService.getRecentMessages(sampleHistory, 3);
console.log(historyService.formatHistory(recent));
console.log("");

console.log("4. メッセージ統計:");
const stats = historyService.countMessages(sampleHistory);
console.log(`総数: ${stats.total}, ユーザー: ${stats.user}, Bot: ${stats.bot}`);
```

---

## 11. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] if/elseで条件分岐ができる
- [ ] 比較演算子（===, !==, <, >など）と論理演算子（&&, ||, !）が使える
- [ ] forとforEachで繰り返し処理ができる
- [ ] filterで配列から条件に合う要素を取り出せる
- [ ] mapで配列の各要素を変換できる
- [ ] コマンド判定の仕組みが理解できた

### 次回予告

**第5回: 非同期処理と await (最大の山)**

次回は、この講座で最も重要な「非同期処理」を学びます。

- 同期処理と非同期処理の違い
- Promiseの概念
- async/await の使い方
- なぜawaitを書き忘れると事故が起きるのか

---

## 12. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: filterとmapの違いは？

<details>
<summary>答えの例</summary>

filterは「条件に合う要素だけを取り出す」（要素数が減る可能性がある）。  
mapは「各要素を変換する」（要素数は変わらない）。  
例: [1,2,3,4,5]に対して、filter(偶数)は[2,4]、map(2倍)は[2,4,6,8,10]になる。

</details>

### Q2: なぜコマンド判定を関数に分けるの？

<details>
<summary>答えの例</summary>

新しいコマンドを追加するとき、commandHandlerのif文とcommandServiceの処理関数を追加するだけで済む。  
全部1箇所に書くと、コードが長くなって読みにくくなり、修正も大変になる。

</details>

### Q3: forとforEachはどう使い分ける？

<details>
<summary>答えの例</summary>

配列の全要素を順番に処理するだけならforEachが読みやすい。  
途中でbreakしたい、逆順に処理したい、複雑な条件がある場合はforを使う。

</details>

---

これらの質問に答えられれば、第4回は完璧です！

次回、[第5回: 非同期処理と await (最大の山)](./lesson05.md) でお会いしましょう。

---

**学習メモ**: 条件分岐が複雑になったら、switch文も検討できます。ただし、まずはif/elseで十分です。
