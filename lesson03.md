# 第3回: 関数と処理の分解 (Bot構造の基礎)

## この回で学ぶこと

- 関数とは何か（処理をまとめる仕組み）
- 引数と戻り値（関数に値を渡す・受け取る）
- なぜ処理を分けるのか（保守性・再利用性）
- handler と service の役割分担
- module.exports と require の仕組み

## ゴール

この回を終えたあなたは、**「この関数は何担当か」が分かる**ようになります。

---

## 1. 関数とは何か？

### 関数の本質：「名前をつけた処理のまとまり」

関数とは、**複数の処理をまとめて、名前をつけたもの**です。

```javascript
// 関数を定義する
function greet() {
  console.log("こんにちは！");
  console.log("今日もいい天気ですね。");
}

// 関数を呼び出す
greet();
```

### 実行結果

```
こんにちは！
今日もいい天気ですね。
```

### なぜ関数が必要なのか？

関数がないと、同じ処理を何度も書く必要があります。

```javascript
// 関数なしの場合
console.log("こんにちは！");
console.log("今日もいい天気ですね。");

console.log("こんにちは！");
console.log("今日もいい天気ですね。");

console.log("こんにちは！");
console.log("今日もいい天気ですね。");

// 関数ありの場合
function greet() {
  console.log("こんにちは！");
  console.log("今日もいい天気ですね。");
}

greet();
greet();
greet();
```

**関数を使うと、同じ処理を何度も書かずに済む**のです。

---

## 2. 引数（ひきすう）とは？

### 引数の本質：「関数に渡す値」

引数を使うと、**関数に値を渡して、動作を変えられます**。

```javascript
// 引数を受け取る関数
function greet(name) {
  console.log(`こんにちは、${name}さん！`);
}

// 異なる引数で呼び出す
greet("太郎");  // こんにちは、太郎さん！
greet("花子");  // こんにちは、花子さん！
greet("次郎");  // こんにちは、次郎さん！
```

### 複数の引数を受け取る

```javascript
function introduce(name, age) {
  console.log(`${name}さんは${age}歳です。`);
}

introduce("太郎", 25);  // 太郎さんは25歳です。
introduce("花子", 30);  // 花子さんは30歳です。
```

### 引数のイメージ図

```
関数呼び出し: greet("太郎")
                     ↓
              引数として渡される
                     ↓
関数定義: function greet(name)
                           ↓
                      nameに"太郎"が入る
                           ↓
                  処理の中で使える
```

---

## 3. 戻り値（もどりち）とは？

### 戻り値の本質：「関数が返す結果」

戻り値を使うと、**関数の実行結果を受け取れます**。

```javascript
// 戻り値を返す関数
function add(a, b) {
  return a + b;
}

// 戻り値を受け取る
const result = add(3, 5);
console.log(result);  // 8
```

### returnの役割

```javascript
function greet(name) {
  return `こんにちは、${name}さん！`;
}

// 戻り値を変数に格納
const message = greet("太郎");
console.log(message);  // こんにちは、太郎さん！

// 戻り値を直接使う
console.log(greet("花子"));  // こんにちは、花子さん！
```

### returnの重要なルール

**returnの後の処理は実行されない**

```javascript
function example() {
  console.log("これは実行される");
  return "終了";
  console.log("これは実行されない");  // ←ここには到達しない
}

example();
```

---

## 4. なぜ処理を分けるのか？

### 理由1: 保守性（メンテナンスしやすい）

処理が1つの関数にまとまっていると、**修正箇所が明確**になります。

```javascript
// ❌ 処理が散らばっている
console.log("=== Bot起動 ===");
console.log("Discordに接続中...");
console.log("接続完了！");

// 別の場所で同じ処理
console.log("=== Bot起動 ===");
console.log("Discordに接続中...");
console.log("接続完了！");

// ✅ 関数にまとめる
function logBotStart() {
  console.log("=== Bot起動 ===");
  console.log("Discordに接続中...");
  console.log("接続完了！");
}

logBotStart();
logBotStart();
```

**メッセージを変更したいとき、関数1つを修正すれば全体に反映される**

### 理由2: 再利用性（同じ処理を何度も使える）

```javascript
// メッセージを整形する関数
function formatMessage(user, text) {
  return `[${user}] ${text}`;
}

// 色々な場所で使える
console.log(formatMessage("太郎", "おはよう"));
console.log(formatMessage("花子", "こんにちは"));
console.log(formatMessage("次郎", "こんばんは"));
```

### 理由3: 可読性（コードが読みやすい）

```javascript
// ❌ 全部ベタ書き
const message = await fetch(apiUrl)
  .then(res => res.json())
  .then(data => data.candidates[0].content.parts[0].text);

// ✅ 関数に分ける
async function fetchAIResponse(apiUrl) {
  const response = await fetch(apiUrl);
  const data = await response.json();
  return extractText(data);
}

function extractText(data) {
  return data.candidates[0].content.parts[0].text;
}

const message = await fetchAIResponse(apiUrl);
```

**何をしているか一目で分かる**

---

## 5. Discord Botの処理構造

Discord Botは、大きく分けて3つの層で構成されます。

```
┌─────────────────────────────────┐
│   index.js (起動ファイル)        │
│   - Botの起動と設定              │
│   - イベントリスナーの登録        │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│   handler (処理の振り分け)       │
│   - メッセージの判定             │
│   - コマンドの振り分け           │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│   service (実際の処理)           │
│   - DB操作                      │
│   - API呼び出し                 │
│   - データ加工                  │
└─────────────────────────────────┘
```

### それぞれの役割

| 層 | 役割 | 例 |
|---|---|---|
| index.js | Botの起動と全体管理 | Discordへの接続、イベント受信 |
| handler | 処理の振り分け | 「このコマンドならこの処理」 |
| service | 実際の処理 | DBへの保存、APIへのリクエスト |

---

## 6. handler と service の役割分担

### handlerの役割：「交通整理」

handlerは、**どの処理を実行するか判断する**役割です。

```javascript
// messageHandler.js
async function handleMessage(message) {
  // Bot自身のメッセージは無視
  if (message.author.bot) return;

  // コマンドの判定
  if (message.content.startsWith('!help')) {
    await handleHelpCommand(message);
  } else if (message.content.startsWith('!clear')) {
    await handleClearCommand(message);
  } else {
    await handleNormalMessage(message);
  }
}
```

### serviceの役割：「実際の作業」

serviceは、**具体的な処理を実行する**役割です。

```javascript
// geminiService.js
async function generateAIResponse(userMessage, history) {
  // Gemini APIへのリクエスト
  const response = await callGeminiAPI(userMessage, history);
  
  // レスポンスからテキストを抽出
  const text = extractText(response);
  
  return text;
}
```

### なぜ分けるのか？

```
例: レストランの仕組み

ホール係（handler） → 「何を注文するか」判断
                     → 「どの料理人に頼むか」振り分け

料理人（service）  → 「実際に料理を作る」

この分担により、それぞれが自分の仕事に集中できる
```

---

## 7. module.exports と require の仕組み

### module.exportsの役割：「外部に公開」

`module.exports`を使うと、**他のファイルから関数を使えるようにできます**。

```javascript
// utils.js
function formatDate(date) {
  return date.toISOString();
}

function formatMessage(user, text) {
  return `[${user}] ${text}`;
}

// 外部に公開
module.exports = {
  formatDate,
  formatMessage
};
```

### requireの役割：「外部から読み込み」

`require`を使うと、**他のファイルの関数を読み込めます**。

```javascript
// main.js
const utils = require('./utils.js');

// utils.jsの関数を使える
const message = utils.formatMessage("太郎", "おはよう");
console.log(message);  // [太郎] おはよう
```

### データの流れ

```
utils.js
  ↓ module.exports で公開
main.js
  ↓ require で読み込み
使える！
```

---

## 8. 実践：簡単な Bot処理を作る

### プロジェクトフォルダの作成

```
my-discord-bot/
├── index.js
├── handlers/
│   └── messageHandler.js
└── services/
    └── replyService.js
```

### ステップ1: replyService.js（実際の処理）

**ファイル**: `my-discord-bot/services/replyService.js`

```javascript
// 返信を生成する処理
function generateReply(userMessage) {
  // シンプルなルールベースの返信
  if (userMessage.includes("こんにちは")) {
    return "こんにちは！調子はどうですか？";
  } else if (userMessage.includes("ありがとう")) {
    return "どういたしまして！";
  } else {
    return "そうなんですね。もっと聞かせてください！";
  }
}

// メッセージをフォーマット
function formatReply(username, reply) {
  return `${username}さんへ: ${reply}`;
}

// 外部に公開
module.exports = {
  generateReply,
  formatReply
};
```

### ステップ2: messageHandler.js（振り分け処理）

**ファイル**: `my-discord-bot/handlers/messageHandler.js`

```javascript
// replyService.js を読み込む
const replyService = require('../services/replyService.js');

// メッセージを処理する関数
async function handleMessage(message) {
  // Bot自身のメッセージは無視
  if (message.author.bot) {
    return;
  }

  // ユーザー名とメッセージ内容を取得
  const username = message.author.username;
  const userMessage = message.content;

  console.log(`受信: [${username}] ${userMessage}`);

  // 返信を生成（serviceを使う）
  const reply = replyService.generateReply(userMessage);
  
  // 返信をフォーマット（serviceを使う）
  const formattedReply = replyService.formatReply(username, reply);

  console.log(`送信: ${formattedReply}`);

  // 実際にDiscordに送信（今回は省略）
  // await message.reply(formattedReply);
}

// 外部に公開
module.exports = {
  handleMessage
};
```

### ステップ3: index.js（起動ファイル）

**ファイル**: `my-discord-bot/index.js`

```javascript
// messageHandler.js を読み込む
const messageHandler = require('./handlers/messageHandler.js');

// テスト用のメッセージオブジェクト
const testMessage1 = {
  author: {
    bot: false,
    username: "太郎"
  },
  content: "こんにちは"
};

const testMessage2 = {
  author: {
    bot: false,
    username: "花子"
  },
  content: "ありがとう"
};

const testMessage3 = {
  author: {
    bot: false,
    username: "次郎"
  },
  content: "今日はいい天気ですね"
};

// メッセージを処理
console.log("=== Bot処理テスト ===\n");

messageHandler.handleMessage(testMessage1);
console.log("");

messageHandler.handleMessage(testMessage2);
console.log("");

messageHandler.handleMessage(testMessage3);
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
=== Bot処理テスト ===

受信: [太郎] こんにちは
送信: 太郎さんへ: こんにちは！調子はどうですか？

受信: [花子] ありがとう
送信: 花子さんへ: どういたしまして！

受信: [次郎] 今日はいい天気ですね
送信: 次郎さんへ: そうなんですね。もっと聞かせてください！
```

---

## 9. 処理の流れを追ってみよう

### データの流れ

```
1. index.js
   ↓ testMessage1を渡す
   
2. messageHandler.handleMessage(testMessage1)
   ↓ メッセージ内容"こんにちは"を取り出す
   
3. replyService.generateReply("こんにちは")
   ↓ "こんにちは！調子はどうですか？"を返す
   
4. replyService.formatReply("太郎", "こんにちは！調子はどうですか？")
   ↓ "太郎さんへ: こんにちは！調子はどうですか？"を返す
   
5. console.log("送信: 太郎さんへ: こんにちは！調子はどうですか？")
```

### 各層の責任

| ファイル | 層 | 責任 |
|---------|---|-----|
| index.js | 起動 | Botの起動、テストメッセージの準備 |
| messageHandler.js | handler | メッセージの受け取り、処理の振り分け |
| replyService.js | service | 返信の生成、フォーマット |

---

## 10. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── index.js
├── handlers/
│   └── messageHandler.js
└── services/
    └── replyService.js
```

### replyService.js（完成版）

**ファイルの場所**: `my-discord-bot/services/replyService.js`

```javascript
// 返信を生成する処理
function generateReply(userMessage) {
  // シンプルなルールベースの返信
  if (userMessage.includes("こんにちは")) {
    return "こんにちは！調子はどうですか？";
  } else if (userMessage.includes("ありがとう")) {
    return "どういたしまして！";
  } else if (userMessage.includes("おやすみ")) {
    return "おやすみなさい！良い夢を！";
  } else {
    return "そうなんですね。もっと聞かせてください！";
  }
}

// メッセージをフォーマット
function formatReply(username, reply) {
  const timestamp = new Date().toLocaleTimeString();
  return `[${timestamp}] ${username}さんへ: ${reply}`;
}

// 外部に公開
module.exports = {
  generateReply,
  formatReply
};
```

### messageHandler.js（完成版）

**ファイルの場所**: `my-discord-bot/handlers/messageHandler.js`

```javascript
// replyService.js を読み込む
const replyService = require('../services/replyService.js');

// メッセージを処理する関数
async function handleMessage(message) {
  // Bot自身のメッセージは無視
  if (message.author.bot) {
    console.log("Bot自身のメッセージなので無視します");
    return;
  }

  // ユーザー名とメッセージ内容を取得
  const username = message.author.username;
  const userMessage = message.content;

  console.log(`📨 受信: [${username}] ${userMessage}`);

  // 返信を生成（serviceを使う）
  const reply = replyService.generateReply(userMessage);
  
  // 返信をフォーマット（serviceを使う）
  const formattedReply = replyService.formatReply(username, reply);

  console.log(`📤 送信: ${formattedReply}`);

  // 実際にDiscordに送信（今回は省略）
  // await message.reply(formattedReply);
}

// 外部に公開
module.exports = {
  handleMessage
};
```

### index.js（完成版）

**ファイルの場所**: `my-discord-bot/index.js`

```javascript
// messageHandler.js を読み込む
const messageHandler = require('./handlers/messageHandler.js');

// テスト用のメッセージオブジェクト
const testMessages = [
  {
    author: { bot: false, username: "太郎" },
    content: "こんにちは"
  },
  {
    author: { bot: false, username: "花子" },
    content: "ありがとう"
  },
  {
    author: { bot: false, username: "次郎" },
    content: "今日はいい天気ですね"
  },
  {
    author: { bot: false, username: "四郎" },
    content: "おやすみ"
  },
  {
    author: { bot: true, username: "Bot" },
    content: "これはBotのメッセージ"
  }
];

// メッセージを処理
console.log("=== Bot処理テスト開始 ===\n");

testMessages.forEach((message, index) => {
  console.log(`--- テスト ${index + 1} ---`);
  messageHandler.handleMessage(message);
  console.log("");
});

console.log("=== Bot処理テスト終了 ===");
```

---

## 11. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] 関数は「処理をまとめて名前をつけたもの」
- [ ] 引数は「関数に渡す値」、戻り値は「関数が返す結果」
- [ ] 処理を分ける理由は「保守性・再利用性・可読性」
- [ ] handlerは「交通整理」、serviceは「実際の作業」
- [ ] module.exportsで公開、requireで読み込む

### 次回予告

**第4回: 条件分岐と繰り返し (Botが判断する理由)**

次回は、「Botがどう判断するか」を学びます。

- if/else による条件分岐
- for/forEach による繰り返し処理
- 配列のフィルタリング
- コマンド判定の仕組み

---

## 12. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: handlerとserviceの違いは？

<details>
<summary>答えの例</summary>

handlerは「どの処理を実行するか判断する交通整理役」。  
serviceは「実際の処理を実行する作業員」。  
この分担により、それぞれが自分の責任に集中でき、コードが整理される。

</details>

### Q2: なぜ関数に分けるの？

<details>
<summary>答えの例</summary>

1. 保守性：修正箇所が明確になる  
2. 再利用性：同じ処理を何度も使える  
3. 可読性：何をしているか一目で分かる  

全部を1つの場所に書くと、どこを修正すればいいか分からなくなる。

</details>

### Q3: module.exportsは何をしているの？

<details>
<summary>答えの例</summary>

あるファイルの関数を、他のファイルから使えるようにする仕組み。  
module.exportsで公開し、requireで読み込むことで、処理を複数のファイルに分割できる。

</details>

---

これらの質問に答えられれば、第3回は完璧です！

次回、[第4回: 条件分岐と繰り返し (Botが判断する理由)](./lesson04.md) でお会いしましょう。

---

**学習メモ**: 関数の引数と戻り値が混乱しやすい場合は、実際に手を動かして色々な値を渡してみましょう。
