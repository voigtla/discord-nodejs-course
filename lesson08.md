# 第8回: 外部APIとGemini APIの理解

## この回で学ぶこと

- APIとHTTPリクエストの本質（通信の仕組み）
- Gemini APIの構造とレスポンス
- APIキーの秘匿方法（なぜ流出が危険か）
- エラーハンドリングとリトライ戦略
- レート制限とコスト管理

## ゴール

この回を終えたあなたは、**AI呼び出しを"黒魔術扱いしない"**ようになります。

---

## 1. APIとは何か？

### APIの本質：「プログラム同士の窓口」

API（Application Programming Interface）は、**あるプログラムが別のプログラムの機能を使うための窓口**です。

### 日常の例：レストラン

```
【レストラン = API】
客（あなたのBot） → 注文（リクエスト） → ウェイター（API）
                                        ↓
                                    厨房（Googleのサーバー）
                                        ↓
客（あなたのBot） ← 料理（レスポンス） ← ウェイター（API）
```

**客は厨房の中身を知らなくても、料理を受け取れる**

### Web APIの仕組み

```
あなたのBot → HTTPリクエスト → Gemini APIサーバー
                                    ↓
                              AI処理（内部）
                                    ↓
あなたのBot ← HTTPレスポンス ← Gemini APIサーバー
```

---

## 2. HTTPリクエストとレスポンス

### HTTPとは？

HTTP（HyperText Transfer Protocol）は、**Web上でデータをやり取りするためのルール**です。

### リクエストの構造

```
POST https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent
↑    ↑                                                                              ↑
メソッド  ドメイン                                                                   パス

Headers（ヘッダー）:
  Content-Type: application/json
  Authorization: Bearer YOUR_API_KEY

Body（本文）:
  {
    "contents": [
      { "role": "user", "parts": [{ "text": "こんにちは" }] }
    ]
  }
```

### 主なHTTPメソッド

| メソッド | 意味 | 用途 |
|---------|------|------|
| GET | 取得 | データを取得するだけ |
| POST | 送信 | データを送信して処理を依頼 |
| PUT | 更新 | データを更新 |
| DELETE | 削除 | データを削除 |

**Gemini APIは、POSTメソッドを使う**

### レスポンスの構造

```
Status Code（ステータスコード）: 200 OK
↑
成功・失敗を示す番号

Headers（ヘッダー）:
  Content-Type: application/json

Body（本文）:
  {
    "candidates": [
      {
        "content": {
          "parts": [{ "text": "こんにちは！どのようなお手伝いができますか？" }]
        }
      }
    ]
  }
```

### 主なステータスコード

| コード | 意味 | 説明 |
|-------|------|------|
| 200 | OK | 成功 |
| 400 | Bad Request | リクエストが不正 |
| 401 | Unauthorized | 認証エラー（APIキーが間違っている） |
| 429 | Too Many Requests | リクエストが多すぎる（レート制限） |
| 500 | Internal Server Error | サーバー側のエラー |

---

## 3. Gemini APIの構造

### Gemini APIとは？

Gemini APIは、**Googleの生成AIを使うためのWeb API**です。

### エンドポイント（接続先）

```
https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=YOUR_API_KEY
```

### リクエストの形式

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        { "text": "こんにちは" }
      ]
    }
  ]
}
```

### 会話履歴を含むリクエスト

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "Pythonとは何ですか？" }]
    },
    {
      "role": "model",
      "parts": [{ "text": "Pythonはプログラミング言語です。" }]
    },
    {
      "role": "user",
      "parts": [{ "text": "初心者にも使いやすいですか？" }]
    }
  ]
}
```

**会話履歴を送ることで、文脈を理解した返答が得られる**

### レスポンスの構造

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "はい、Pythonは初心者にも使いやすい言語として知られています。"
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_HATE_SPEECH",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 15,
    "candidatesTokenCount": 30,
    "totalTokenCount": 45
  }
}
```

### レスポンスから返答を取り出す

```javascript
const response = await fetch(apiUrl, { method: 'POST', body: ... });
const data = await response.json();

// AI返答を取り出す
const aiText = data.candidates[0].content.parts[0].text;
console.log(aiText);
```

---

## 4. APIキーとは何か？

### APIキーの本質：「身分証明書」

APIキーは、**「あなたが正規のユーザーであることを証明するパスワード」**です。

```
例: 銀行のキャッシュカード
→ カード（APIキー）がないとお金を引き出せない
→ カードを盗まれると、他人に使われる
```

### なぜAPIキーが必要なのか？

1. **本人確認**: 誰がAPIを使っているか識別
2. **利用量管理**: ユーザーごとの使用量を追跡
3. **課金**: 使った分だけ請求
4. **悪用防止**: 不正利用を防ぐ

### APIキーの形式

```
例: AIzaSyD1234567890abcdefghijklmnopqrstuvwxyz
     ↑
     ランダムな文字列（40文字程度）
```

---

## 5. APIキーの秘匿方法

### なぜAPIキーを秘匿するのか？

**APIキーが流出すると、他人があなたのアカウントでAPIを使い放題**

```
悪用例:
1. あなたのAPIキーがGitHubに公開される
2. 悪意のある第三者がAPIキーを発見
3. 大量のリクエストを送信（あなたの請求書に）
4. 月末に高額請求が来る
```

### ❌ ダメな例：コードに直接書く

```javascript
// ❌ 絶対にやってはいけない
const apiKey = "AIzaSyD1234567890abcdefghijklmnopqrstuvwxyz";
```

このコードをGitHubにpushすると、**数分以内にボットが検知して悪用される**

### ✅ 正しい方法：環境変数で管理

**ステップ1: .envファイルに書く**

```env
GEMINI_API_KEY=AIzaSyD1234567890abcdefghijklmnopqrstuvwxyz
```

**ステップ2: .gitignoreに追加**

```
.env
```

**ステップ3: コードから読み込む**

```javascript
require('dotenv').config();
const apiKey = process.env.GEMINI_API_KEY;
```

### .gitignoreの重要性

```gitignore
# 環境変数
.env

# データベース
*.db

# ログファイル
*.log

# node_modules
node_modules/
```

**これらのファイルはGitに含めない**

---

## 6. エラーハンドリング

### APIエラーの種類

| エラー | 原因 | 対処法 |
|-------|------|--------|
| 400 Bad Request | リクエスト形式が間違っている | リクエストを修正 |
| 401 Unauthorized | APIキーが間違っている | APIキーを確認 |
| 429 Too Many Requests | リクエストが多すぎる | 待ってからリトライ |
| 500 Internal Server Error | サーバー側のエラー | 少し待ってからリトライ |

### エラーハンドリングの実装

```javascript
async function callGeminiAPI(prompt) {
  try {
    const response = await fetch(apiUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        contents: [
          { role: 'user', parts: [{ text: prompt }] }
        ]
      })
    });
    
    // ステータスコードをチェック
    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`API Error ${response.status}: ${errorData.error.message}`);
    }
    
    const data = await response.json();
    return data.candidates[0].content.parts[0].text;
    
  } catch (error) {
    console.error('API呼び出しエラー:', error.message);
    throw error;
  }
}
```

---

## 7. レート制限とリトライ戦略

### レート制限とは？

レート制限は、**「1分間に何回までリクエストできるか」の制限**です。

```
例: Gemini API無料枠
→ 1分間に60リクエストまで
→ 超えると 429 エラー
```

### リトライ戦略（Exponential Backoff）

```javascript
async function callWithRetry(apiFunction, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await apiFunction();
    } catch (error) {
      // 429エラー（レート制限）の場合のみリトライ
      if (error.message.includes('429') && i < maxRetries - 1) {
        const waitTime = Math.pow(2, i) * 1000;  // 1秒、2秒、4秒...
        console.log(`レート制限に達しました。${waitTime}ms待機します...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      } else {
        throw error;  // それ以外はエラーを投げる
      }
    }
  }
}
```

---

## 8. 実践：Gemini APIの実装

### プロジェクトフォルダの作成

```
my-discord-bot/
├── .env
├── .gitignore
├── gemini-api.js
└── test-gemini.js
```

### ステップ1: .envファイルを作成

**ファイル**: `my-discord-bot/.env`

```env
GEMINI_API_KEY=あなたのAPIキー
```

**注意**: 実際のAPIキーに置き換えてください

### ステップ2: .gitignoreを作成

**ファイル**: `my-discord-bot/.gitignore`

```
.env
*.db
node_modules/
*.log
```

### ステップ3: gemini-api.js（Gemini API操作）

**ファイル**: `my-discord-bot/gemini-api.js`

```javascript
require('dotenv').config();

const GEMINI_API_KEY = process.env.GEMINI_API_KEY;
const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`;

// Gemini APIを呼び出す
async function callGeminiAPI(prompt, conversationHistory = []) {
  // 会話履歴を構築
  const contents = [
    ...conversationHistory,
    {
      role: 'user',
      parts: [{ text: prompt }]
    }
  ];
  
  const requestBody = {
    contents: contents,
    generationConfig: {
      temperature: 0.7,
      maxOutputTokens: 1000
    }
  };
  
  try {
    console.log('🔄 Gemini APIにリクエスト送信中...');
    
    const response = await fetch(GEMINI_API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
    });
    
    // ステータスコードをチェック
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      const errorMessage = errorData.error?.message || `HTTP ${response.status}`;
      throw new Error(`API Error: ${errorMessage}`);
    }
    
    const data = await response.json();
    
    // レスポンスから返答を取り出す
    if (!data.candidates || data.candidates.length === 0) {
      throw new Error('APIからの返答がありません');
    }
    
    const aiText = data.candidates[0].content.parts[0].text;
    console.log('✅ Gemini APIから返答を受信');
    
    return {
      success: true,
      text: aiText,
      usage: data.usageMetadata
    };
    
  } catch (error) {
    console.error('❌ API呼び出しエラー:', error.message);
    return {
      success: false,
      error: error.message
    };
  }
}

// リトライ機能付きのAPI呼び出し
async function callGeminiAPIWithRetry(prompt, conversationHistory = [], maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    const result = await callGeminiAPI(prompt, conversationHistory);
    
    if (result.success) {
      return result;
    }
    
    // 429エラー（レート制限）の場合のみリトライ
    if (result.error.includes('429') && i < maxRetries - 1) {
      const waitTime = Math.pow(2, i) * 1000;
      console.log(`⏳ レート制限に達しました。${waitTime / 1000}秒待機します...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    } else {
      // それ以外のエラーはリトライしない
      return result;
    }
  }
  
  return {
    success: false,
    error: 'リトライ回数の上限に達しました'
  };
}

// 会話履歴を整形
function formatConversationHistory(messages) {
  return messages.map(msg => ({
    role: msg.role,
    parts: [{ text: msg.content }]
  }));
}

// 外部に公開
module.exports = {
  callGeminiAPI,
  callGeminiAPIWithRetry,
  formatConversationHistory
};
```

### ステップ4: test-gemini.js（テスト）

**ファイル**: `my-discord-bot/test-gemini.js`

```javascript
const gemini = require('./gemini-api.js');

console.log("=== Gemini APIテスト ===\n");

// テスト1: シンプルな質問
async function test1() {
  console.log("【テスト1】シンプルな質問\n");
  
  const result = await gemini.callGeminiAPI('JavaScriptとは何ですか？');
  
  if (result.success) {
    console.log('AI返答:');
    console.log(result.text);
    console.log('\n使用トークン:', result.usage);
  } else {
    console.log('エラー:', result.error);
  }
  
  console.log('\n' + '='.repeat(60) + '\n');
}

// テスト2: 会話履歴を含む質問
async function test2() {
  console.log("【テスト2】会話履歴を含む質問\n");
  
  // 会話履歴
  const history = gemini.formatConversationHistory([
    { role: 'user', content: 'Pythonとは何ですか？' },
    { role: 'model', content: 'Pythonはプログラミング言語です。シンプルで読みやすい文法が特徴です。' }
  ]);
  
  const result = await gemini.callGeminiAPI('初心者にも使いやすいですか？', history);
  
  if (result.success) {
    console.log('AI返答:');
    console.log(result.text);
  } else {
    console.log('エラー:', result.error);
  }
  
  console.log('\n' + '='.repeat(60) + '\n');
}

// テスト3: リトライ機能のテスト
async function test3() {
  console.log("【テスト3】リトライ機能付きAPI呼び出し\n");
  
  const result = await gemini.callGeminiAPIWithRetry('Discord Botの作り方を教えてください');
  
  if (result.success) {
    console.log('AI返答:');
    console.log(result.text.substring(0, 200) + '...');  // 最初の200文字だけ表示
  } else {
    console.log('エラー:', result.error);
  }
  
  console.log('\n' + '='.repeat(60) + '\n');
}

// すべてのテストを実行
async function runAllTests() {
  try {
    await test1();
    await test2();
    await test3();
    
    console.log('=== すべてのテスト完了 ===');
  } catch (error) {
    console.error('テスト中にエラーが発生:', error);
  }
}

runAllTests();
```

### 実行方法

```bash
# プロジェクトフォルダに移動
cd my-discord-bot

# dotenvをインストール
npm install dotenv

# 実行
node test-gemini.js
```

### 実行結果

```
=== Gemini APIテスト ===

【テスト1】シンプルな質問

🔄 Gemini APIにリクエスト送信中...
✅ Gemini APIから返答を受信
AI返答:
JavaScriptは、Webページに動的な要素を追加するために使用されるプログラミング言語です。ブラウザ上で動作し、ユーザーのアクションに応じてコンテンツを変更したり、アニメーションを作成したりできます。

使用トークン: { promptTokenCount: 8, candidatesTokenCount: 65, totalTokenCount: 73 }

============================================================

【テスト2】会話履歴を含む質問

🔄 Gemini APIにリクエスト送信中...
✅ Gemini APIから返答を受信
AI返答:
はい、Pythonは初心者にも非常に使いやすい言語として知られています。英語に近い文法で書けるため、プログラミングの概念を学びやすいです。

============================================================

【テスト3】リトライ機能付きAPI呼び出し

🔄 Gemini APIにリクエスト送信中...
✅ Gemini APIから返答を受信
AI返答:
Discord Botを作るには、まずDiscord Developer Portalでアプリケーションを作成し、Botトークンを取得します。次に、Node.jsとdiscord.jsライブラリを使ってコードを書きます...

============================================================

=== すべてのテスト完了 ===
```

---

## 9. 完成版ソースコード（この回の結論）

### ファイル構成

```
my-discord-bot/
├── .env
├── .gitignore
├── gemini-api.js
└── test-gemini.js
```

### .env（完成版）

**ファイルの場所**: `my-discord-bot/.env`

```env
GEMINI_API_KEY=あなたのAPIキー
```

### .gitignore（完成版）

**ファイルの場所**: `my-discord-bot/.gitignore`

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

### gemini-api.js（完成版）

**上記の「ステップ3」のコードと同じ**

### test-gemini.js（完成版）

**上記の「ステップ4」のコードと同じ**

---

## 10. まとめ：この回で理解すべきこと

### ✅ チェックリスト

- [ ] APIは「プログラム同士の窓口」
- [ ] HTTPリクエストとレスポンスの構造を理解した
- [ ] Gemini APIのリクエスト/レスポンス形式を理解した
- [ ] APIキーは「身分証明書」で、流出すると悪用される
- [ ] .envファイルと.gitignoreでAPIキーを秘匿する
- [ ] エラーハンドリングとリトライ戦略が必要

### APIキーを守る3つのルール

1. **.envファイルに書く**（コードに直接書かない）
2. **.gitignoreに追加**（Gitに含めない）
3. **絶対に公開しない**（スクリーンショットにも注意）

### 次回予告

**第9回: セキュリティ・エラー・安全設計**

次回は、「やってはいけないこと集」を学びます。

- try/catchの真意
- バリデーション（入力値チェック）
- APIキー流出経路
- スパム対策

---

## 11. 言葉で説明してみよう

次の質問に、自分の言葉で答えてみてください。

### Q1: なぜAPIキーを.envファイルで管理するの？

<details>
<summary>答えの例</summary>

APIキーをコードに直接書くと、GitHubにpushしたときに公開されてしまう。  
.envファイルに書いて.gitignoreに追加すれば、Gitの管理対象外になるので安全。  
また、環境ごとに異なるAPIキーを使い分けられる（開発用/本番用など）。

</details>

### Q2: レート制限はなぜあるの？

<details>
<summary>答えの例</summary>

サーバーへの負荷を分散し、すべてのユーザーが公平にAPIを使えるようにするため。  
また、悪意のある大量リクエスト（DDoS攻撃など）を防ぐため。  
レート制限があることで、無料枠でも安定してサービスを提供できる。

</details>

### Q3: Exponential Backoffとは？

<details>
<summary>答えの例</summary>

リトライ時の待機時間を指数関数的に増やす戦略。  
1回目: 1秒待つ、2回目: 2秒待つ、3回目: 4秒待つ...  
これにより、サーバーへの負荷を減らしつつ、エラーから回復できる。

</details>

---

これらの質問に答えられれば、第8回は完璧です！

次回、[第9回: セキュリティ・エラー・安全設計](./lesson09.md) でお会いしましょう。

---

**学習メモ**: APIキーが流出したら、すぐにAPIキーを再生成してください。古いキーは無効化されます。
