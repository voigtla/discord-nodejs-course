# 第5回: 非同期処理と await (最大の山)

## この回で学ぶこと

- 同期処理と非同期処理の違い（「待つ」とは何か）
- Promiseの概念（「未来の値」を表すオブジェクト）
- async/awaitの使い方（非同期処理を同期的に書く魔法）
- awaitを書き忘れるとなぜ事故が起きるのか
- Discord BotとAPIにおける非同期処理

## ⚠️ 重要度：最高

この回は、**Discord Bot開発で最も重要な内容**です。  
非同期処理を理解しないと、以下のような問題が起きます：

- Botが返信する前にメッセージを送ろうとする
- データベースに保存される前に次の処理が始まる
- APIの返答を待たずに「undefined」を表示する

じっくり時間をかけて理解してください。

## ゴール

この回を終えたあなたは、**awaitを理由付きで書ける**ようになります。

---

## 1. 同期処理とは？

### 同期処理の本質：「順番に実行される」

同期処理は、**前の処理が終わるまで次の処理が待つ**仕組みです。

```javascript
console.log("1. 処理開始");
console.log("2. 計算中...");
const result = 10 + 20;
console.log("3. 結果: " + result);
console.log("4. 処理終了");
```

### 実行結果

```
1. 処理開始
2. 計算中...
3. 結果: 30
4. 処理終了
```

**順番通りに実行される** → これが同期処理

---

## 2. 非同期処理とは？

### 非同期処理の本質：「待たずに次に進む」

非同期処理は、**時間がかかる処理を「後回し」にして、先に次の処理を実行する**仕組みです。

### 日常の例：レストラン

```
【同期処理（1人ずつ対応）】
客A: 注文 → 待つ → 料理受け取り（完了）
       ↓ Aが完了するまでBは注文できない
客B: 注文 → 待つ → 料理受け取り（完了）

【非同期処理（同時に対応）】
客A: 注文 → 番号札を受け取る
客B: 注文 → 番号札を受け取る  ← Aを待たずに注文できる
     ↓
客A: 料理ができたら呼ばれる
客B: 料理ができたら呼ばれる
```

### プログラミングにおける非同期処理

```javascript
console.log("1. 処理開始");

// 3秒後に実行される（非同期）
setTimeout(() => {
  console.log("3. 3秒後の処理");
}, 3000);

console.log("2. すぐに実行される");
```

### 実行結果

```
1. 処理開始
2. すぐに実行される
（3秒待つ）
3. 3秒後の処理
```

**2が3より先に実行される** → これが非同期処理

---

## 3. なぜ非同期処理が必要なのか？

### 問題：時間がかかる処理でプログラムが止まる

```javascript
// ❌ 同期的にAPIを呼ぶと...（実際には動かない疑似コード）
console.log("1. API呼び出し開始");
const response = callAPI();  // ← 3秒かかる
console.log("2. API呼び出し完了");

// この間、プログラムは完全に停止
// 他のユーザーのメッセージも処理できない
```

### 解決：非同期処理で他の作業を並行して行う

```javascript
console.log("1. API呼び出し開始");

// APIを呼んでいる間、他の処理ができる
callAPIAsync().then((response) => {
  console.log("3. API呼び出し完了");
});

console.log("2. 他の処理を実行");
```

### Discord Botで非同期処理が必須な理由

1. **Discord APIへの通信**（メッセージ送信など）
2. **Gemini APIへの通信**（AI返答の生成）
3. **データベースへのアクセス**（履歴の保存・取得）

これらはすべて**時間がかかる処理**なので、非同期で行う必要があります。

---

## 4. Promiseとは？

### Promiseの本質：「未来の値を表すオブジェクト」

Promiseは、**「今はまだないけど、後で手に入る値」**を表します。

```javascript
// レストランの例
const order = new Promise((resolve, reject) => {
  console.log("料理を作っています...");
  
  setTimeout(() => {
    const dish = "カレーライス";
    resolve(dish);  // 料理ができた！
  }, 3000);
});

console.log("番号札を受け取りました");

// 料理ができたら実行
order.then((dish) => {
  console.log(`料理が来ました: ${dish}`);
});
```

### 実行結果

```
料理を作っています...
番号札を受け取りました
（3秒待つ）
料理が来ました: カレーライス
```

### Promiseの3つの状態

```
pending（保留中）   → まだ結果が出ていない
   ↓
fulfilled（成功）  → 処理が成功した（resolve）
   または
rejected（失敗）   → 処理が失敗した（reject）
```

---

## 5. async/await：非同期処理を同期的に書く

### awaitの本質：「結果が出るまで待つ」

`await`を使うと、**非同期処理の結果が出るまで待つ**ことができます。

```javascript
// Promiseを返す関数
function wait(seconds) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(`${seconds}秒待ちました`);
    }, seconds * 1000);
  });
}

// async関数の中でawaitを使う
async function example() {
  console.log("1. 開始");
  
  const result = await wait(2);  // ← 2秒待つ
  console.log("2. " + result);
  
  console.log("3. 終了");
}

example();
```

### 実行結果

```
1. 開始
（2秒待つ）
2. 2秒待ちました
3. 終了
```

**awaitを使うと、順番通りに実行される**

---

## 6. awaitを書き忘れるとどうなるか？

### 問題：awaitがないと「Promise」オブジェクトが返る

```javascript
function fetchData() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve("重要なデータ");
    }, 1000);
  });
}

// ❌ awaitなし
async function badExample() {
  const data = fetchData();  // ← awaitを忘れた
  console.log(data);  // Promise { <pending> }
  console.log(data.toUpperCase());  // エラー！
}

badExample();
```

### 実行結果

```
Promise { <pending> }
TypeError: data.toUpperCase is not a function
```

### 解決：awaitを使う

```javascript
// ✅ awaitあり
async function goodExample() {
  const data = await fetchData();  // ← awaitで待つ
  console.log(data);  // "重要なデータ"
  console.log(data.toUpperCase());  // "重要なデータ"
}

goodExample();
```

### 実行結果

```
（1秒待つ）
重要なデータ
重要なデータ
```

---

## 7. Discord Botでの非同期処理

### メッセージ送信は非同期

```javascript
// ❌ awaitなし（よくあるミス）
async function handleMessage(message) {
  message.reply("こんにちは");  // ← awaitがない
  console.log("返信しました");  // ← 返信完了前に実行される
}

// ✅ awaitあり
async function handleMessage(message) {
  await message.reply("こんにちは");  // ← 返信完了まで待つ
  console.log("返信しました");  // ← 返信完了後に実行される
}
```

### データベース操作は非同期

```javascript
// ❌ awaitなし
async function saveMessage(text) {
  const result = db.insert(text);  // ← Promiseが返る
  console.log("保存完了");  // ← 保存完了前に実行される
  return result;  // ← Promise { <pending> }
}

// ✅ awaitあり
async function saveMessage(text) {
  const result = await db.insert(text);  // ← 保存完了まで待つ
  console.log("保存完了");  // ← 保存完了後に実行される
  return result;  // ← 実際の結果
}
```

### API呼び出しは非同期

```javascript
// ❌ awaitなし
async function getAIResponse(prompt) {
  const response = fetch(apiUrl, { body: prompt });  // ← Promiseが返る
  const data = response.json();  // ← エラー！responseはまだPromise
  return data.text;
}

// ✅ awaitあり
async function getAIResponse(prompt) {
  const response = await fetch(apiUrl, { body: prompt });  // ← 完了まで待つ
  const data = await response.json();  // ← 完了まで待つ
  return data.text;
}
```

---

## 8. 複数の非同期処理を順番に実行

### 逐次実行：1つずつ順番に待つ

```javascript
async function sequential() {
  console.log("開始");
  
  const result1 = await wait(1);  // 1秒待つ
  console.log(result1);
  
  const result2 = await wait(1);  // さらに1秒待つ
  console.log(result2);
  
  console.log("終了");
}

sequential();
```

### 実行結果

```
開始
（1秒待つ）
1秒待ちました
（1秒待つ）
1秒待ちました
終了
（合計2秒）
```

### 並列実行：同時に実行して全部待つ

```javascript
async function parallel() {
  console.log("開始");
  
  // 同時に開始
  const promise1 = wait(1);
  const promise2 = wait(1);
  
  // 両方の完了を待つ
  const results = await Promise.all([promise1, promise2]);
  
  console.log(results);  // ["1秒待ちました", "1秒待ちました"]
  console.log("終了");
}

parallel();
```

### 実行結果

```
開始
（1秒待つ）
["1秒待ちました", "1秒待ちました"]
終了
（合計1秒）
```

---

## 9. エラーハンドリング：try/catch

### 非同期処理でもエラーは起きる

```javascript
function riskyOperation() {
  return new Promise((resolve, reject) => {
    const success = Math.random() > 0.5;
    
    setTimeout(() => {
      if (success) {
        resolve("成功！");
      } else {
        reject(new Error("失敗..."));
      }
    }, 1000);
  });
}

// ❌ エラーハンドリングなし
async function badExample() {
  const result = await riskyOperation();  // ← エラーが起きたら停止
  console.log(result);
}

// ✅ try/catchでエラーを捕捉
async function goodExample() {
  try {
    const result = await riskyOperation();
    console.log(result);
  } catch (error) {
    console.log("エラーが発生:", error.message);
  }
}

goodExample();
```

---

## 10. 実践：非同期処理を含むBot処理

### プロジェクトフォルダの作成

```
my-discord-bot/
├── index.js
├── services/
│   ├── apiService.js
│   └── dbService.js
└── handlers/
    └── asyncHandler.js
```

### ステップ1: apiService.js（API呼び出しシミュレーション）

**ファイル**: `my-discord-bot/services/apiService.js`

```javascript
// Gemini APIを呼ぶシミュレーション
function callGeminiAPI(prompt) {
  return new Promise((resolve, reject) => {
    console.log("  → API呼び出し開始...");
    
    // 2秒後に返答
    setTimeout(() => {
      if (prompt) {
        const response = `AI返答: 「${prompt}」について考えました。`;
        console.log("  → API呼び出し完了");
        resolve(response);
      } else {
        reject(new Error("プロンプトが空です"));
      }
    }, 2000);
  });
}

// 外部に公開
module.exports = {
  callGeminiAPI
};
```

### ステップ2: dbService.js（DB操作シミュレーション）

**ファイル**: `my-discord-bot/services/dbService.js`

```javascript
// データベースに保存するシミュレーション
function saveToDatabase(username, message) {
  return new Promise((resolve) => {
    console.log("  → DB保存開始...");
    
    // 1秒後に保存完了
    setTimeout(() => {
      const record = {
        id: Date.now(),
        username: username,
        message: message,
        savedAt: new Date().toISOString()
      };
      console.log("  → DB保存完了");
      resolve(record);
    }, 1000);
  });
}

// データベースから取得するシミュレーション
function loadFromDatabase(username) {
  return new Promise((resolve) => {
    console.log("  → DB取得開始...");
    
    // 1秒後に取得完了
    setTimeout(() => {
      const history = [
        `${username}: 過去のメッセージ1`,
        `${username}: 過去のメッセージ2`
      ];
      console.log("  → DB取得完了");
      resolve(history);
    }, 1000);
  });
}

// 外部に公開
module.exports = {
  saveToDatabase,
  loadFromDatabase
};
```

### ステップ3: asyncHandler.js（非同期処理の統合）

**ファイル**: `my-discord-bot/handlers/asyncHandler.js`

```javascript
// サービスを読み込む
const apiService = require('../services/apiService.js');
const dbService = require('../services/dbService.js');

// メッセージを処理する（すべて非同期）
async function handleAsyncMessage(message) {
  const username = message.author.username;
  const content = message.content;
  
  console.log(`\n[${username}] ${content}`);
  console.log("処理開始...\n");
  
  try {
    // ステップ1: 過去の履歴を取得（非同期）
    console.log("ステップ1: 履歴取得");
    const history = await dbService.loadFromDatabase(username);
    console.log(`  履歴: ${history.length}件\n`);
    
    // ステップ2: AI APIを呼び出し（非同期）
    console.log("ステップ2: AI返答生成");
    const aiResponse = await apiService.callGeminiAPI(content);
    console.log(`  返答: ${aiResponse}\n`);
    
    // ステップ3: 今回のやり取りを保存（非同期）
    console.log("ステップ3: やり取りを保存");
    const record = await dbService.saveToDatabase(username, content);
    console.log(`  保存ID: ${record.id}\n`);
    
    console.log("処理完了！");
    return aiResponse;
    
  } catch (error) {
    console.error("エラーが発生:", error.message);
    return "エラーが発生しました。";
  }
}

// 外部に公開
module.exports = {
  handleAsyncMessage
};
```

### ステップ4: index.js（起動ファイル）

**ファイル**: `my-discord-bot/index.js`

```javascript
// ハンドラを読み込む
const asyncHandler = require('./handlers/asyncHandler.js');

// テスト用のメッセージ
const testMessage = {
  author: {
    username: "太郎"
  },
  content: "おすすめの本を教えて"
};

// 非同期処理のテスト
console.log("=== 非同期処理テスト ===");

// async関数を実行
async function runTest() {
  const response = await asyncHandler.handleAsyncMessage(testMessage);
  console.log("\n最終返答:", response);
}

runTest();
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
=== 非同期処理テスト ===

[太郎] おすすめの本を教えて
処理開始...

ステップ1: 履歴取得
  → DB取得開始...
（1秒待つ）
  → DB取得完了
  履歴: 2件

ステップ2: AI返答生成
  → API呼び出し開始...
（2秒待つ）
  → API呼び出し完了
  返答: AI返答: 「おすすめの本を教えて」について考えました。

ステップ3: やり取りを保存
  → DB保存開始...
（1秒待つ）
  → DB保存完了
  保存ID: 1706889123456

処理完了！

最終返答: AI返答: 「おすすめの本を教えて」について考えました。

（合計実行時間: 約4秒）
```

---

## 11. awaitを書き忘れた場合の比較

### ❌ awaitなしの場合

**ファイル**: `bad-example.js`

```javascript
const apiService = require('./services/apiService.js');
const dbService = require('./services/dbService.js');

// ❌ awaitを書き忘れた例
async function badHandler(message) {
  console.log("処理開始\n");
  
  // awaitがない！
  const history = dbService.loadFromDatabase(message.author.username);
  console.log("履歴:", history);  // Promise { <pending> }
  
  const aiResponse = apiService.callGeminiAPI(message.content);
  console.log("返答:", aiResponse);  // Promise { <pending> }
  
  const record = dbService.saveToDatabase(message.author.username, message.content);
  console.log("保存:", record);  // Promise { <pending> }
  
  console.log("\n処理完了");
  return aiResponse;  // Promiseが返る（値ではない）
}

const testMessage = {
  author: { username: "太郎" },
  content: "テスト"
};

badHandler(testMessage).then((response) => {
  console.log("\n最終返答:", response);  // Promise { <pending> }
});
```

### 実行結果

```
処理開始

履歴: Promise { <pending> }
  → DB取得開始...
返答: Promise { <pending> }
  → API呼び出し開始...
保存: Promise { <pending> }
  → DB保存開始...

処理完了
最終返答: Promise { <pending> }

（その後、バックグラウンドで処理が完了する）
  → DB取得完了
  → API呼び出し完了
  → DB保存完了
```

**問題点**:
- 値が取得できていない（すべてPromise）
- 処理の順序が保証されない
- エラーハンドリングができない

---

## 12. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── index.js
├── handlers/
│   └── asyncHandler.js
└── services/
    ├── apiService.js
    └── dbService.js
```

### apiService.js（完成版）

**ファイルの場所**: `my-discord-bot/services/apiService.js`

```javascript
// Gemini APIを呼ぶシミュレーション
function callGeminiAPI(prompt) {
  return new Promise((resolve, reject) => {
    console.log("  → API呼び出し開始...");
    
    setTimeout(() => {
      if (prompt) {
        const response = `AI返答: 「${prompt}」について考えました。`;
        console.log("  → API呼び出し完了");
        resolve(response);
      } else {
        console.log("  → API呼び出し失敗");
        reject(new Error("プロンプトが空です"));
      }
    }, 2000);
  });
}

module.exports = {
  callGeminiAPI
};
```

### dbService.js（完成版）

**ファイルの場所**: `my-discord-bot/services/dbService.js`

```javascript
// データベースに保存するシミュレーション
function saveToDatabase(username, message) {
  return new Promise((resolve) => {
    console.log("  → DB保存開始...");
    
    setTimeout(() => {
      const record = {
        id: Date.now(),
        username: username,
        message: message,
        savedAt: new Date().toISOString()
      };
      console.log("  → DB保存完了");
      resolve(record);
    }, 1000);
  });
}

// データベースから取得するシミュレーション
function loadFromDatabase(username) {
  return new Promise((resolve) => {
    console.log("  → DB取得開始...");
    
    setTimeout(() => {
      const history = [
        `${username}: 過去のメッセージ1`,
        `${username}: 過去のメッセージ2`
      ];
      console.log("  → DB取得完了");
      resolve(history);
    }, 1000);
  });
}

module.exports = {
  saveToDatabase,
  loadFromDatabase
};
```

### asyncHandler.js（完成版）

**ファイルの場所**: `my-discord-bot/handlers/asyncHandler.js`

```javascript
const apiService = require('../services/apiService.js');
const dbService = require('../services/dbService.js');

// メッセージを処理する（すべて非同期）
async function handleAsyncMessage(message) {
  const username = message.author.username;
  const content = message.content;
  
  console.log(`\n[${username}] ${content}`);
  console.log("処理開始...\n");
  
  try {
    // ステップ1: 過去の履歴を取得（await必須）
    console.log("ステップ1: 履歴取得");
    const history = await dbService.loadFromDatabase(username);
    console.log(`  履歴: ${history.length}件\n`);
    
    // ステップ2: AI APIを呼び出し（await必須）
    console.log("ステップ2: AI返答生成");
    const aiResponse = await apiService.callGeminiAPI(content);
    console.log(`  返答: ${aiResponse}\n`);
    
    // ステップ3: 今回のやり取りを保存（await必須）
    console.log("ステップ3: やり取りを保存");
    await dbService.saveToDatabase(username, content);
    await dbService.saveToDatabase("Bot", aiResponse);
    console.log("  保存完了\n");
    
    console.log("処理完了！");
    return aiResponse;
    
  } catch (error) {
    console.error("\nエラーが発生:", error.message);
    return "エラーが発生しました。もう一度お試しください。";
  }
}

module.exports = {
  handleAsyncMessage
};
```

### index.js（完成版）

**ファイルの場所**: `my-discord-bot/index.js`

```javascript
const asyncHandler = require('./handlers/asyncHandler.js');

// テスト用のメッセージ
const testMessages = [
  {
    author: { username: "太郎" },
    content: "おすすめの本を教えて"
  },
  {
    author: { username: "花子" },
    content: "今日の天気は？"
  }
];

// 非同期処理のテスト
console.log("=== 非同期処理テスト ===");

// 複数のメッセージを順番に処理
async function runTests() {
  for (const message of testMessages) {
    const response = await asyncHandler.handleAsyncMessage(message);
    console.log(`\n最終返答: ${response}`);
    console.log("\n" + "=".repeat(50));
  }
  
  console.log("\nすべてのテスト完了");
}

runTests();
```

---

## 13. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] 同期処理は「順番に実行」、非同期処理は「待たずに次へ」
- [ ] Promiseは「未来の値」を表すオブジェクト
- [ ] awaitは「非同期処理の結果が出るまで待つ」
- [ ] awaitを忘れると、Promise { <pending> } が返る
- [ ] Discord/API/DBの操作はすべて非同期
- [ ] try/catchでエラーハンドリングが必要

### 重要なルール

**「Promiseを返す関数を呼ぶときは、必ずawaitを付ける」**

これを忘れると、99%のバグはこれが原因です。

### 次回予告

**第6回: SQLite基礎 (なぜDBを使うのか)**

次回は、「データベース」の基礎を学びます。

- SQLiteとは何か
- ファイル保存との違い
- テーブル・行・列の概念
- SELECT/INSERTの基本

---

## 14. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: awaitを書き忘れるとなぜ問題なの？

<details>
<summary>答えの例</summary>

awaitを書かないと、「Promise { <pending> }」というオブジェクトが返ってくる。  
これは「まだ結果が出ていない」という意味で、実際の値ではない。  
この状態で次の処理に進むと、undefinedエラーが起きたり、処理の順序が狂ったりする。

</details>

### Q2: Promise.allは何をするの？

<details>
<summary>答えの例</summary>

複数の非同期処理を同時に実行して、すべての完了を待つ。  
逐次実行（await → await → await）だと時間がかかるが、  
Promise.allを使えば並列実行できるので、合計時間を短縮できる。

</details>

### Q3: なぜDiscord Botは非同期処理だらけなの？

<details>
<summary>答えの例</summary>

Botは複数のユーザーから同時にメッセージを受け取る。  
各メッセージの処理（Discord通信、API呼び出し、DB操作）には時間がかかる。  
同期処理だと、1人の処理中に他のユーザーが待たされてしまう。  
非同期処理なら、複数のユーザーを同時に対応できる。

</details>

---

これらの質問に答えられれば、第5回は完璧です！

次回、[第6回: SQLite基礎 (なぜDBを使うのか)](./lesson06.md) でお会いしましょう。

---

**学習メモ**: async/awaitが難しく感じたら、何度もコードを実行して、awaitのあり/なしで結果がどう変わるか確認しましょう。
