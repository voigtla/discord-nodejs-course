# 第2回: 変数・型・JSON (設定とデータの正体)

## この回で学ぶこと

- 変数とは何か（データに名前をつける箱）
- 型とは何か（データの種類：number / string / boolean）
- object と array（複雑なデータの表現方法）
- JSONの正体（データの共通フォーマット）
- .envファイルと設定値の読み込み

## ゴール

この回を終えたあなたは、**設定ファイルとレスポンスJSONが読める**ようになります。

---

## 1. 変数とは何か？

### 変数の本質：「名前をつけた箱」

プログラミングにおいて、**変数とはデータに名前をつけて保存する仕組み**です。

```javascript
let botName = "MyBot";
```

これは、「`MyBot`という文字列データを、`botName`という名前の箱に入れた」という意味です。

### なぜ変数が必要なのか？

変数がないと、同じデータを何度も書く必要があります。

```javascript
// 変数なしの場合
console.log("MyBot が起動しました");
console.log("MyBot が接続中です");
console.log("MyBot の準備完了");

// 変数ありの場合
let botName = "MyBot";
console.log(botName + " が起動しました");
console.log(botName + " が接続中です");
console.log(botName + " の準備完了");
```

変数を使うと、**名前を変更したいときに1箇所変えるだけで済む**のです。

### 変数の宣言方法

JavaScriptには3つの宣言方法があります。

```javascript
let userName = "太郎";      // 再代入可能（値を後で変更できる）
const maxUsers = 100;       // 再代入不可（値を変更できない）
var oldStyle = "古い書き方"; // 古い書き方（使わない）
```

**推奨**: 基本的に`const`を使い、変更が必要な場合だけ`let`を使う

### 変数名のルール

```javascript
// ✅ 良い例
let userName = "太郎";
let messageCount = 5;
let isReady = true;

// ❌ ダメな例
let user name = "太郎";  // スペースは使えない
let 2users = 10;        // 数字から始まる名前は使えない
let user-name = "太郎"; // ハイフンは使えない
```

---

## 2. 型とは何か？

### 型の本質：「データの種類」

プログラミングにおいて、**型とはデータの種類**を指します。

JavaScriptには以下の基本的な型があります。

### (1) number（数値型）

```javascript
let age = 25;
let price = 1000;
let temperature = -5.5;
```

**特徴**: 整数も小数も、すべて`number`型

### (2) string（文字列型）

```javascript
let name = "太郎";
let message = 'こんにちは';
let greeting = `Hello, ${name}!`; // テンプレートリテラル（変数埋め込み）
```

**特徴**: `"`、`'`、`` ` ``（バッククォート）のいずれかで囲む

### (3) boolean（真偽値型）

```javascript
let isActive = true;
let hasError = false;
```

**特徴**: `true`（真）か`false`（偽）の2つのみ

### 型の確認方法

```javascript
console.log(typeof 123);        // "number"
console.log(typeof "こんにちは"); // "string"
console.log(typeof true);       // "boolean"
```

---

## 3. object（オブジェクト）: 複数のデータをまとめる

### objectの本質：「名前付きの複数の値」

objectは、**複数のデータを1つにまとめる**ための型です。

```javascript
const user = {
  name: "太郎",
  age: 25,
  isAdmin: false
};
```

これは、「`user`というobjectに、`name`、`age`、`isAdmin`という3つのプロパティがある」という意味です。

### objectの読み書き

```javascript
// 読み取り
console.log(user.name);  // "太郎"
console.log(user.age);   // 25

// 書き込み
user.age = 26;
console.log(user.age);   // 26
```

### なぜobjectが必要なのか？

objectがないと、関連するデータがバラバラになります。

```javascript
// objectなしの場合
let userName = "太郎";
let userAge = 25;
let userIsAdmin = false;

// objectありの場合
const user = {
  name: "太郎",
  age: 25,
  isAdmin: false
};
```

**objectを使うと、関連するデータを1つにまとめられる**のです。

---

## 4. array（配列）: 複数の値を順番に並べる

### arrayの本質：「番号付きの複数の値」

arrayは、**複数の値を順番に並べる**ための型です。

```javascript
const fruits = ["りんご", "バナナ", "オレンジ"];
```

これは、「`fruits`というarrayに、3つの文字列が順番に入っている」という意味です。

### arrayの読み書き

```javascript
// 読み取り（インデックスは0から始まる）
console.log(fruits[0]);  // "りんご"
console.log(fruits[1]);  // "バナナ"
console.log(fruits[2]);  // "オレンジ"

// 要素数の取得
console.log(fruits.length);  // 3

// 追加
fruits.push("ぶどう");
console.log(fruits);  // ["りんご", "バナナ", "オレンジ", "ぶどう"]
```

### objectとarrayの違い

```javascript
// object: 名前でアクセス
const user = {
  name: "太郎",
  age: 25
};
console.log(user.name);  // "太郎"

// array: 番号でアクセス
const users = ["太郎", "花子", "次郎"];
console.log(users[0]);   // "太郎"
```

**使い分け**: 
- 名前付きのデータ → object
- 順番のあるリスト → array

---

## 5. JSONとは何か？

### JSONの本質：「データの共通フォーマット」

JSON（JavaScript Object Notation）は、**異なるプログラム間でデータをやり取りするための標準フォーマット**です。

### JSONの例

```json
{
  "name": "太郎",
  "age": 25,
  "hobbies": ["読書", "音楽", "プログラミング"],
  "address": {
    "city": "東京",
    "zip": "100-0001"
  }
}
```

### JSONのルール

1. **キー（名前）は必ずダブルクォーテーションで囲む**
   - ❌ `{name: "太郎"}` （JavaScriptではOKだが、JSONではNG）
   - ✅ `{"name": "太郎"}`

2. **文字列はダブルクォーテーションのみ**
   - ❌ `{"name": '太郎'}` （シングルクォーテーションはNG）
   - ✅ `{"name": "太郎"}`

3. **末尾のカンマは禁止**
   - ❌ `{"name": "太郎", "age": 25,}` （最後のカンマはNG）
   - ✅ `{"name": "太郎", "age": 25}`

### JavaScriptとJSONの変換

```javascript
// JavaScriptのobject
const user = {
  name: "太郎",
  age: 25
};

// objectをJSON文字列に変換
const jsonString = JSON.stringify(user);
console.log(jsonString);  // '{"name":"太郎","age":25}'

// JSON文字列をobjectに変換
const parsedUser = JSON.parse(jsonString);
console.log(parsedUser.name);  // "太郎"
```

### なぜJSONが重要なのか？

Discord BotとGemini APIのやり取りは、すべて**JSON形式**で行われます。

```
あなたのBot → JSONでリクエスト → Gemini API
                                    ↓
あなたのBot ← JSONでレスポンス ← Gemini API
```

JSONが読めると、**APIが何を返しているか理解できる**のです。

---

## 6. .envファイルと設定値

### .envファイルとは？

**.envファイルは、秘密情報や設定値を保存するファイル**です。

### .envファイルの例

```env
DISCORD_TOKEN=あなたのDiscordトークン
GEMINI_API_KEY=あなたのGemini APIキー
BOT_PREFIX=!
```

### なぜ.envファイルを使うのか？

理由1: **秘密情報をコードに直接書かない**

```javascript
// ❌ ダメな例（トークンが丸見え）
const token = "MTE1NzM2OTk4ODU5MDUyNDQxNw.GxYzAB.secret";

// ✅ 良い例（.envから読み込む）
const token = process.env.DISCORD_TOKEN;
```

理由2: **設定を簡単に変更できる**

コードを変更せず、.envファイルだけ編集すれば設定を変更できます。

### .envファイルの読み込み

Node.jsでは、`dotenv`というライブラリを使います。

```javascript
// .envファイルを読み込む
require('dotenv').config();

// 環境変数にアクセス
const discordToken = process.env.DISCORD_TOKEN;
const geminiApiKey = process.env.GEMINI_API_KEY;

console.log(discordToken);  // .envに書いた値が表示される
```

---

## 7. Discord Bot設定ファイルの実例

### package.json（プロジェクト設定）

```json
{
  "name": "my-discord-bot",
  "version": "1.0.0",
  "description": "Discord Bot with Gemini AI",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "discord.js": "^14.14.1",
    "dotenv": "^16.4.1",
    "@google/generative-ai": "^0.1.3",
    "better-sqlite3": "^9.3.0"
  }
}
```

### 各項目の意味

- `name`: プロジェクト名
- `version`: バージョン番号
- `main`: 最初に実行されるファイル
- `scripts`: コマンドのショートカット（`npm start`で起動）
- `dependencies`: 必要なライブラリのリスト

---

## 8. Gemini APIレスポンスの読み方

### 実際のレスポンス例

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "こんにちは！今日はどんなことをお手伝いしましょうか？"
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP"
    }
  ]
}
```

### データ構造の読み解き

```javascript
// レスポンスを変数に格納
const response = {
  candidates: [
    {
      content: {
        parts: [
          { text: "こんにちは！今日はどんなことをお手伝いしましょうか？" }
        ],
        role: "model"
      },
      finishReason: "STOP"
    }
  ]
};

// AI返答を取り出す
const aiText = response.candidates[0].content.parts[0].text;
console.log(aiText);  // "こんにちは！今日はどんなことをお手伝いしましょうか？"
```

### 階層構造の理解

```
response (object)
 └─ candidates (array)
     └─ [0] (object)
         └─ content (object)
             └─ parts (array)
                 └─ [0] (object)
                     └─ text (string) ← ここに返答が入っている
```

---

## 9. 実践：簡単な設定ファイルを作る

### プロジェクトフォルダの作成

まず、以下のフォルダ構成を作ります。

```
my-discord-bot/
├── .env
├── config.js
└── test.js
```

### ステップ1: .envファイルを作成

**ファイル**: `my-discord-bot/.env`

```env
BOT_NAME=MyBot
BOT_PREFIX=!
MAX_HISTORY=10
```

### ステップ2: config.jsを作成

**ファイル**: `my-discord-bot/config.js`

```javascript
// .envファイルを読み込む
require('dotenv').config();

// 設定値をobjectにまとめる
const config = {
  botName: process.env.BOT_NAME || "DefaultBot",
  botPrefix: process.env.BOT_PREFIX || "!",
  maxHistory: parseInt(process.env.MAX_HISTORY) || 10
};

// 他のファイルから使えるようにエクスポート
module.exports = config;
```

### ステップ3: test.jsで設定を読み込む

**ファイル**: `my-discord-bot/test.js`

```javascript
// config.jsから設定を読み込む
const config = require('./config.js');

// 設定値を表示
console.log("Bot名:", config.botName);
console.log("コマンド接頭辞:", config.botPrefix);
console.log("最大履歴数:", config.maxHistory);
```

### 実行方法

```bash
# my-discord-botフォルダに移動
cd my-discord-bot

# dotenvライブラリをインストール
npm install dotenv

# 実行
node test.js
```

### 実行結果

```
Bot名: MyBot
コマンド接頭辞: !
最大履歴数: 10
```

---

## 10. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── .env
├── config.js
└── test.js
```

### .env（完成版）

**ファイルの場所**: `my-discord-bot/.env`

```env
BOT_NAME=MyBot
BOT_PREFIX=!
MAX_HISTORY=10
```

### config.js（完成版）

**ファイルの場所**: `my-discord-bot/config.js`

```javascript
// .envファイルを読み込む
require('dotenv').config();

// 設定値をobjectにまとめる
const config = {
  botName: process.env.BOT_NAME || "DefaultBot",
  botPrefix: process.env.BOT_PREFIX || "!",
  maxHistory: parseInt(process.env.MAX_HISTORY) || 10
};

// 他のファイルから使えるようにエクスポート
module.exports = config;
```

### test.js（完成版）

**ファイルの場所**: `my-discord-bot/test.js`

```javascript
// config.jsから設定を読み込む
const config = require('./config.js');

// 設定値を表示
console.log("=== Bot設定 ===");
console.log("Bot名:", config.botName);
console.log("コマンド接頭辞:", config.botPrefix);
console.log("最大履歴数:", config.maxHistory);

// 設定値を使った例
const exampleCommand = config.botPrefix + "help";
console.log("\nコマンド例:", exampleCommand);

// objectとarrayの例
const botInfo = {
  name: config.botName,
  version: "1.0.0",
  features: ["AI対話", "履歴保存", "コマンド実行"]
};

console.log("\nBot情報:", JSON.stringify(botInfo, null, 2));
```

---

## 11. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] 変数は「データに名前をつけた箱」である
- [ ] 型には number / string / boolean がある
- [ ] objectは「名前付きの複数の値」、arrayは「番号付きの複数の値」
- [ ] JSONは「データの共通フォーマット」で、APIとのやり取りに使う
- [ ] .envファイルで秘密情報と設定値を管理する

### 次回予告

**第3回: 関数と処理の分解 (Bot構造の基礎)**

次回は、「処理を部品に分ける」方法を学びます。

- 関数とは何か（処理をまとめる仕組み）
- 引数と戻り値（関数に値を渡す・受け取る）
- handler と service の役割分担
- なぜ処理を分けるのか

---

## 12. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: objectとarrayの違いは？

<details>
<summary>答えの例</summary>

objectは「名前でアクセスする複数の値」、arrayは「番号でアクセスする複数の値」。  
関連する項目をまとめたいときはobject、順番のあるリストを作りたいときはarrayを使う。

</details>

### Q2: なぜ.envファイルを使うの？

<details>
<summary>答えの例</summary>

秘密情報（APIキーやトークン）をコードに直接書くと、GitHubに公開したときに流出する危険がある。  
.envファイルに書いて.gitignoreで除外すれば、秘密情報を守れる。  
また、設定値を.envファイルに集約すれば、コードを変更せずに設定を変更できる。

</details>

### Q3: JSONとJavaScriptのobjectの違いは？

<details>
<summary>答えの例</summary>

JSONは「テキスト形式のデータ交換フォーマット」で、厳格なルールがある（キーはダブルクォーテーション必須など）。  
JavaScriptのobjectは「プログラム内で使うデータ構造」で、より柔軟（キーはクォーテーション不要など）。  
APIとやり取りするときはJSON、プログラム内で扱うときはobjectを使う。

</details>

---

これらの質問に答えられれば、第2回は完璧です！

次回、[第3回: 関数と処理の分解 (Bot構造の基礎)](./lesson03.md) でお会いしましょう。

---

**学習メモ**: JSONの階層構造が分かりにくかったら、実際のAPIレスポンスをコピーして、1つずつ階層を辿ってみましょう。
