# 英語学習チャットシステム API設計書

## 1. 概要
本システムは **Next.js App Router** を採用し、データアクセスには主に **Server Actions** を、AIのストリーミングレスポンスには **Route Handlers** を使用する。
認証には **Supabase Auth** を利用し、すべてのAPI/Action呼び出しにおいてサーバーサイドで認証状態を検証する。

## 2. API エンドポイント (Route Handlers)

Vercel AI SDKを利用したチャット生成機能のためのエンドポイント。

### 2.1 チャットメッセージ生成

*   **URL:** `/api/chat`
*   **Method:** `POST`
*   **概要:** ユーザーの入力メッセージを受け取り、AIからの応答をストリーミング形式で返す。また、処理プロセス内でメッセージ（ユーザー・AI双方）をDBに永続化する。

#### Request
*   **Content-Type:** `application/json`
*   **Body:**
    ```json
    {
      "messages": [
        {
          "role": "user",
          "content": "こんにちは"
        },
        ...
      ]
    }
    ```
    ※ Vercel AI SDKの `useChat` フックが送信する標準フォーマット。

#### Response
*   **Content-Type:** `text/event-stream` (または `text/plain`)
*   **Body:** AI生成テキストのストリーム

#### 内部処理フロー (Server Side)
1.  **認証チェック:** Supabase Authによりユーザーを特定。未認証の場合は401エラー。
2.  **ユーザーメッセージ保存:** リクエスト内の最新のユーザーメッセージを `ChatMessages` テーブルに保存。
3.  **AI生成リクエスト:**
    *   システムプロンプトを含めてLLM (OpenAI/Gemini等) へリクエスト。
    *   **指示内容:** 「入力に対して、1. Formal, 2. Casual, 3. Native の3つの異なるニュアンスで回答すること」および「JSON形式または特定の区切り文字を用いたフォーマット」を指定し、構造化データとして扱えるように生成させる（またはFunction Callingを利用）。
4.  **ストリーミング応答:** 生成されたテキストをクライアントへストリーミング。
5.  **完了後処理 (onFinish):**
    *   AIの応答テキストを解析し、3つのメッセージ（Formal, Casual, Native）に分割。
    *   `ChatMessages` テーブルにAIのメッセージとして3レコードを保存（`parent_message_id` には手順2で保存したIDを設定）。

---

## 3. Server Actions (Data Access)

クライアントコンポーネントから直接呼び出し可能な非同期関数群。`app/actions/` ディレクトリに配置することを想定。

### 3.1 チャット履歴取得 (`getChatHistory`)

*   **概要:** 過去のチャット履歴を取得する。
*   **引数:** なし (SessionからユーザーIDを特定)
*   **戻り値:** `Promise<ChatMessage[]>`
    ```typescript
    type ChatMessage = {
      id: string;
      role: 'user' | 'assistant'; // DBの 'USER'|'AI' を変換
      content: string;
      responseType?: 'formal' | 'casual' | 'native';
      createdAt: Date;
    };
    ```

### 3.2 ブックマーク追加 (`addBookmark`)

*   **概要:** 特定のメッセージ（フレーズ）をブックマークに保存する。
*   **引数:**
    *   `messageId`: string (対象のメッセージID)
    *   `phrase`: string (フレーズ本文)
    *   `meaning`: string (任意: 解説や和訳)
*   **戻り値:** `Promise<{ success: boolean, data?: Bookmark, error?: string }>`

### 3.3 ブックマーク削除 (`deleteBookmark`)

*   **概要:** ブックマークを削除する。
*   **引数:**
    *   `bookmarkId`: string
*   **戻り値:** `Promise<{ success: boolean, error?: string }>`

### 3.4 ブックマーク一覧取得 (`getBookmarks`)

*   **概要:** ユーザーのブックマーク一覧を取得する。
*   **引数:** なし
*   **戻り値:** `Promise<Bookmark[]>`
    ```typescript
    type Bookmark = {
      id: string;
      sourceMessageId: string;
      phrase: string;
      meaning?: string;
      createdAt: Date;
    };
    ```

### 3.5 テキスト読み上げ (`synthesizeSpeech`) ※オプション

*   **概要:** Web Speech APIで不足がある場合、サーバーサイドでTTSを行うためのAction。
*   **引数:**
    *   `text`: string
*   **戻り値:** `Promise<string>` (Base64エンコードされた音声データ、または音声ファイルのURL)
*   **備考:** 開発初期はブラウザ標準の `Web Speech API` を使用するため、必須ではない。

---

## 4. エラーハンドリング

*   **認証エラー:** 全てのエンドポイント/Actionで未認証アクセスをチェックし、認証されていない場合はエラーを返す。
*   **DBエラー:** Supabaseへのクエリ失敗時は、適切なエラーメッセージをクライアントに返す。
*   **AI生成エラー:** LLMのRate LimitやAPIダウン時は、ユーザーにリトライを促すメッセージを表示する。
